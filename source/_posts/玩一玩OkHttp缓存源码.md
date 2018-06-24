---
title: 玩一玩OkHttp缓存源码
toc: true
date: 2018-06-24 17:22:44
tags: [OkHttp, 源码, 缓存, Retrofit, RxJava]
---

## 前言
本文重点讲述Http缓存在OkHttp中的实现，同时也分析了Retrofit+ OkHttp是如何实现网络请求的。

**_本文基于OkHttp3.9.0及Retrofit2.3.0_**

**_你猜的没错，本文很长很长_**

了解Http缓存机制再看本文更佳，这里可以看看我的[《不一样的HTTP缓存体验》](http://crazysunj.com/2018/06/24/%E4%B8%8D%E4%B8%80%E6%A0%B7%E7%9A%84HTTP%E7%BC%93%E5%AD%98%E4%BD%93%E9%AA%8C/)。

本文为了更好地理清思路，夹杂其它的源码分析，如RxJava。

## 分析
### 简单使用
上文也讲述了在Android中的使用，这里再简单回顾一遍。

* 设置缓存策略

```
@Headers("Cache-Control: public, max-age=300")//缓存时间为5分钟
@GET("random/data/{type}/{count}")
Flowable<GankioEntity> getGankio(@Path("type") String type, @Path("count") int count);

builder.addInterceptor(new CrazyDailyCacheInterceptor());
builder.addNetworkInterceptor(new CrazyDailyCacheNetworkInterceptor());
```

*  设置缓存目录

```
Cache cache = new Cache(new File(context.getExternalCacheDir(), CacheConstant.CACHE_DIR_API), 20 * 1024 * 1024);
builder.cache(cache);
```

那么，OkHttp是如何拿到Retrofit的Header，又是如何根据我们的缓存策略进行缓存的，又是如何保存在我们的缓存目录。预知后事如何，请看下文分解。

<!--  more-->
### 第一问
Retrofit源码比较短，但是精辟！最核心的肯定是动态代理。这里简单分析是如何拿到Header然后传给OkHttp。

这是我们的常规写法：

```
mGankioService = new Retrofit.Builder()
        .baseUrl(GankioService.HOST)
        .client(mOkHttpClient)
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build().create(GankioService.class);
```

那么主要就是构造方法的client和create方法。跟进：

```
//Retrofit
public Builder client(OkHttpClient client) {
  return callFactory(checkNotNull(client, "client == null"));
}

public Builder callFactory(okhttp3.Call.Factory factory) {
  this.callFactory = checkNotNull(factory, "factory == null");
  return this;
}
```

OK，简单的赋值，再来看看create方法。

```
//Retrofit
public <T> T create(final Class<T> service) {
    ...
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          ...
          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
           ...
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
}
```

loadServiceMethod方法肯定是解析方法参数，然后一堆赋值什么的，不妨进去看看：

```
//Retrofit
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ...
    //这里知道ServiceMethod其实拿到了Retrofit的引用
    result = new ServiceMethod.Builder<>(this, method).build();
    ...
    return result;
}

//ServiceMethod
Builder(Retrofit retrofit, Method method) {
  ...
  this.methodAnnotations = method.getAnnotations();
  ...
}

public ServiceMethod build() {
  ...
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }
  ...
  return new ServiceMethod<>(this);
}

private void parseMethodAnnotation(Annotation annotation) {
  ...
  } else if (annotation instanceof retrofit2.http.Headers) {
    String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
    ...
    headers = parseHeaders(headersToParse);
  ...
}
```

到这里已经结束了，因为已经产生了headers，如何产生以及架构设计并不是我们关心的。Retrofit生产了headers，那么得给OkHttp消费，回顾Retrofit的create方法，Retrofit会把ServiceMethod传入OkHttpCall，而OkHttpCall只是一个网络操作接口，那么其实不难猜到具体的还是ServiceMethod去完成。毕竟ServiceMethod手握Retrofit，而Retrofit又有最重要的OkHttp。而最后一句无非只是适配，比如我们可以用RxJava。

```
//Retrofit
serviceMethod.callAdapter.adapt(okHttpCall)
```
OK，我知道大家很好奇这是怎么回事，其实并不难，如果用RxJava，是不是subscribe貌似就可以进行网络请求了？有看过源码吗？

```
//Observable
public final void subscribe(Observer<? super T> observer) {
    ...   
    subscribeActual(observer);
    ...
}
```

既然是适配，那么核心方法肯定在adapt中，这里以上文添加的适配器RxJava2CallAdapter为例。

```
//RxJava2CallAdapter
@Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);
    ...
    return observable;
}
```

这里挑选同步的查看：

```
//CallExecuteObservable
@Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    ...
    Response<T> response = call.execute();
    ...
}
```
这时候心应该平静下来了，但这只是个开始，我们的核心缓存还并没有讲。我们再回过神来去看看execute方法。其实如果直接用OkHttp就没有这么多事。

```
//OkHttpCall
@Override public Response<T> execute() throws IOException {
    okhttp3.Call call;
    ...
    call = rawCall = createRawCall();
    ...
    return parseResponse(call.execute());
}
```
没有ServiceMethod？这不可能，跟进createRawCall：

```
//OkHttpCall
private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    //serviceMethod.callFactory就是我们的OkHttpClient
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    ...
    return call;
}
```

这两句很重要，第一句是我们分析了半天的header的封装，第二句是我们的主角OkHttp。简单看一下：

```
//ServiceMethod
Request toRequest(@Nullable Object... args) throws IOException {
    //这里把Retrofit注解拿到的headers封装到Request
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);
    ...
    return requestBuilder.build();
}

ServiceMethod(Builder<R, T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    ...
}
```
至此，我们解决了第一个问题，OkHttp是如何拿到Retrofit的Header。

### 第二问
接下来我们来分析第二问，OkHttp是如何根据我们的缓存策略进行缓存的。

链接上文OkHttpCall的execute方法，最后其实调了okhttp3.Call的execute方法。

```
//OkHttpCall
//serviceMethod.callFactory是我们的OkHttpClient，强调多遍
okhttp3.Call call = serviceMethod.callFactory.newCall(request);

//OkHttpClient
@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}

//RealCall
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    ...
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    ...
    return call;
}

@Override public Response execute() throws IOException {
    ...
    Response result = getResponseWithInterceptorChain();
    ...
}
```
execute方法最关键肯定是这里了，得到我们的结果，那么处理请求以及解析等等都在这了。跟进getResponseWithInterceptorChain方法。

```
//RealCall
Response getResponseWithInterceptorChain() throws IOException {
    ...
    interceptors.addAll(client.interceptors());
    ...
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    ...
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
}
```

OK，如果仔细阅读的同学，不难知道originalRequest其实就是serviceMethod封装的Request。

```
Request request = serviceMethod.toRequest(args);
```
在分析拦截器链执行前，我们要搞清楚4个东西。

* client.interceptors()
* CacheInterceptor
* ConnectInterceptor
* client.networkInterceptors()

```
//OkHttpClient
public List<Interceptor> interceptors() {
    return interceptors;
}

public List<Interceptor> networkInterceptors() {
    return networkInterceptors;
}

//OkHttpClient.Builder
public Builder addInterceptor(Interceptor interceptor) {
  ...
  interceptors.add(interceptor);
  return this;
}

public Builder addNetworkInterceptor(Interceptor interceptor) {
  ...
  networkInterceptors.add(interceptor);
  return this;
}

//OkHttpClient
OkHttpClient(Builder builder) {
    ...
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    ...
}
```
OK，我想你已经了解了。而关于CacheInterceptor我们先放放，因为这是主角，ConnectInterceptor是扩展知识，到时候会简单介绍。

我们再来玩玩RealInterceptorChain。

```
//RealInterceptorChain
public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, RealConnection connection, int index, Request request, Call call,
      EventListener eventListener, int connectTimeout, int readTimeout, int writeTimeout) {
    this.interceptors = interceptors;
    ...
}
```
常规操作，一堆赋值。

```
//RealInterceptorChain
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
}

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    ...
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    ...
    return response;
}
```

OK，也很简单，每次取一个拦截器，然后调用其intercept方法，而这个请求链的关键就是在intercept调用了chain.proceed，如果不调用了，也就不会执行下一个拦截器。既然我们已经了解这个机制，就了解了首先它会执行我们设置的Interceptor，然后调用CacheInterceptor，接下来是ConnectInterceptor，最后是NetworkInterceptor，而如何打断，我相信你也清楚了。

我们详细地来看看CacheInterceptor吧。

```
//CacheInterceptor
@Override public Response intercept(Chain chain) throws IOException {
    //cache是我们传进去的client.internalCache()，若不为空，根据请求得到一个候选响应
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    long now = System.currentTimeMillis();
    //再根据cacheCandidate，当前时间戳和请求得到一个CacheStrategy，而CacheStrategy可以返回一个networkRequest和一个cacheResponse
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    ...
    return response;
}
```

下面的一系列代码都跟这几个有疑问的变量挂钩，如果不理解是没法玩了。我们一个一个来解答。

首先是client.internalCache()，

```
//OkHttpClient
InternalCache internalCache() {
    return cache != null ? cache.internalCache : internalCache;
}
```

我们貌似写过这样的代码：

```
Cache cache = new Cache(new File(context.getExternalCacheDir(), CacheConstant.CACHE_DIR_API), 20 * 1024 * 1024);
builder.cache(cache);
```

会不会有关系呢？走进去看看：

```
//OkHttpClient.Builder
public Builder cache(@Nullable Cache cache) {
  this.cache = cache;
  this.internalCache = null;
  return this;
}
```

现在我们要明白的是Cache这个类到底干嘛用的，internalCache这个变量又是怎么产生的。

```
//Cache
Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
}
```

好吧，真相大白，然后再了解下数据结构就差不多了。

```
//Cache
//key是根据请求url进行md5处理过的
String key = key(request.url());

public static String key(HttpUrl url) {
    return ByteString.encodeUtf8(url.toString()).md5().hex();
}
//value，响应数据有关
DiskLruCache.Snapshot snapshot = cache.get(key);
```

再来看看关键的internalCache。

```
//Cache
final InternalCache internalCache = new InternalCache() {
    @Override public Response get(Request request) throws IOException {
      return Cache.this.get(request);
    }
    ...
};
```

好吧，其实都是调Cache的方法，而Cache又是DiskLruCache去处理的。大致了解到这儿。到这儿我们已经了解到cacheCandidate就是我们的本地缓存。

那么，理解CacheStrategy就成了关键点。我们根据这句代码一步一步去了解：

```
//CacheInterceptor
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
```

```
//CacheStrategy.Factory
public Factory(long nowMillis, Request request, Response cacheResponse) {
  this.nowMillis = nowMillis;
  this.request = request;
  this.cacheResponse = cacheResponse;

  if (cacheResponse != null) {
    this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
    this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
    Headers headers = cacheResponse.headers();
    for (int i = 0, size = headers.size(); i < size; i++) {
      String fieldName = headers.name(i);
      String value = headers.value(i);
      if ("Date".equalsIgnoreCase(fieldName)) {
        servedDate = HttpDate.parse(value);
        servedDateString = value;
      } else if ("Expires".equalsIgnoreCase(fieldName)) {
        expires = HttpDate.parse(value);
      } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
        lastModified = HttpDate.parse(value);
        lastModifiedString = value;
      } else if ("ETag".equalsIgnoreCase(fieldName)) {
        etag = value;
      } else if ("Age".equalsIgnoreCase(fieldName)) {
        ageSeconds = HttpHeaders.parseSeconds(value, -1);
      }
    }
  }
}
```

如果看过我写的Http缓存机制[《不一样的HTTP缓存体验》](http://crazysunj.com/2018/06/24/%E4%B8%8D%E4%B8%80%E6%A0%B7%E7%9A%84HTTP%E7%BC%93%E5%AD%98%E4%BD%93%E9%AA%8C/)肯定对这几个很熟悉，Expires、Last-Modified和ETag等等。那么我们可以大胆地猜测这个类是真正的缓存策略管理类。

OK，再看看get方法。

```
//CacheStrategy.Factory
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();
  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    ...
    return new CacheStrategy(null, null);
  }
  return candidate;
}
```

貌似还不能下结论，再走进getCandidate方法：

```
//CacheStrategy.Factory
private CacheStrategy getCandidate() {
  // 本地不存在缓存，返回request为正常请求，响应为空
  if (cacheResponse == null) {
    return new CacheStrategy(request, null);
  }

  // 请求支持https但缓存没有接受TSL握手，返回同无缓存
  if (request.isHttps() && cacheResponse.handshake() == null) {
    return new CacheStrategy(request, null);
  }

  // 请求和相应有一Cache-Control设置为no-store均同无缓存一样
  if (!isCacheable(cacheResponse, request)) {
    return new CacheStrategy(request, null);
  }

  // 请求Cache-Control设置为no-cache或者说请求首部的If-Modified-Since或If-None-Match不为空，返回同无缓存
  CacheControl requestCaching = request.cacheControl();
  if (requestCaching.noCache() || hasConditions(request)) {
    return new CacheStrategy(request, null);
  }

  //如果cache的Cache-Control是immutable，即服务器认定该资源不会变动，那么返回的request为空，响应为缓存响应
  CacheControl responseCaching = cacheResponse.cacheControl();
  if (responseCaching.immutable()) {
    return new CacheStrategy(null, cacheResponse);
  }

  // 根据age或者说Cache-Control的max-age缓存
  long ageMillis = cacheResponseAge();
  ...
  return new CacheStrategy(null, builder.build());
  ...
  //根据ETag，Last-Modified及Date进行缓存
  if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
  } else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
  } else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
  } else {
    return new CacheStrategy(request, null); 
  }
  ...
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

OK，再来看看构造函数：

```
//CacheStrategy
CacheStrategy(Request networkRequest, Response cacheResponse) {
    this.networkRequest = networkRequest;
    this.cacheResponse = cacheResponse;
}
```
是的，就这么简单。再来回顾get方法。

```
//CacheStrategy.Factory
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();
  //如果Cache-Control为only-if-cached，那么CacheStrategy的request和response均为空
  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    ...
    return new CacheStrategy(null, null);
  }
  return candidate;
}
```
跟我们想的一样，这个类遵循Http的缓存机制。现在我们可以继续阅读CacheInterceptor下面的代码了。

```
//CacheInterceptor
@Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    ...
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    ...
    // 两者都为空的时候，我们已经很清楚了，直接用缓存
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // 如果request为空，依然用缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    // 继续走下面的拦截器
    Response networkResponse = null;
    ...
    networkResponse = chain.proceed(networkRequest);
    ...

    // 服务器返回的也是缓存
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // 关于第三问的先记下来
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } 
      ...
    }
    // 服务器返回原始资源
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    if (cache != null) {
      // 符合缓存规则进行缓存
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // 同样是第三问
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }
      ...
    }
    return response;
}
```

到这里Http的缓存机制算是结束了，我们得知是遵循Http协议的，因此大体流程依然可以看我的[《不一样的HTTP缓存体验》](http://crazysunj.com/2018/06/24/%E4%B8%8D%E4%B8%80%E6%A0%B7%E7%9A%84HTTP%E7%BC%93%E5%AD%98%E4%BD%93%E9%AA%8C/)，这里就不画流程图了，偷个懒。

### 第三问
第二问的分析已经为我们铺垫了许多，例如缓存的底层用的是DiskLruCache。

第二问我们记下了如下代码：

```
//CacheInterceptor
//服务器返回的也是缓存，应该是更新缓存的样子
cache.update(cacheResponse, response);

//正常缓存
CacheRequest cacheRequest = cache.put(response);
return cacheWritingResponse(cacheRequest, response);
```

先来看更新缓存：

```
//Cache
void update(Response cached, Response network) {
    Entry entry = new Entry(network);
    DiskLruCache.Snapshot snapshot = ((CacheResponseBody) cached.body()).snapshot;
    DiskLruCache.Editor editor = null;
    try {
      editor = snapshot.edit(); 
      if (editor != null) {
        entry.writeTo(editor);
        editor.commit();
      }
    } catch (IOException e) {
      abortQuietly(editor);
    }
}
```

额，其实是一脸懵逼的。因为不知道Entry和DiskLruCache的关系。可能这个你得再了解一下Okio，到时候我有空会写一篇介绍。反正这里就是会去除原来缓存的内容然后重新写入。

我们最后看一下正常写入吧。

```
//Cache
@Nullable CacheRequest put(Response response) {
    String requestMethod = response.request().method();
    ...
    if (!requestMethod.equals("GET")) {
      // 只缓存get请求
      return null;
    }

    if (HttpHeaders.hasVaryAll(response)) {
      //header包含*不缓存
      return null;
    }

    //写入数据
    Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      entry.writeTo(editor);
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
}
```

可能有同学好奇何时建立缓存文件的。看看这个应该就清除了。

```
//Cache
editor = cache.edit(key(response.request().url()));

//DiskLruCache
public @Nullable Editor edit(String key) throws IOException {
    return edit(key, ANY_SEQUENCE_NUMBER);
}

synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
    ...
    if (entry == null) {
      entry = new Entry(key);
      lruEntries.put(key, entry);
    }
    ..
    return editor;
}

//DiskLruCache.Entry
Entry(String key) {
  ...
  int truncateTo = fileBuilder.length();
  for (int i = 0; i < valueCount; i++) {
    fileBuilder.append(i);
    cleanFiles[i] = new File(directory, fileBuilder.toString());
    fileBuilder.append(".tmp");
    dirtyFiles[i] = new File(directory, fileBuilder.toString());
    fileBuilder.setLength(truncateTo);
  }
}

//DiskLruCache
DiskLruCache(FileSystem fileSystem, File directory, int appVersion, int valueCount, long maxSize,
      Executor executor) {
    ...
    this.directory = directory;
    ...
}

//Cache
Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
}
```
我觉得你已经搞清楚了。

### 扩展

为啥要扩展一下，讲解ConnectInterceptor呢？主要还是这张图：

![](/img/okhttp_interceptor.png)

我们简单过一遍。

```
//ConnectInterceptor
@Override public Response intercept(Chain chain) throws IOException {
    ...
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    ...
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
}

//StreamAllocation
public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    ...
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
    ...
}

private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    ...
    RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);
    ...
}

private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    ...
    selectedRoute = routeSelection.next();
    ...  
    return result;
}

//RouteSelector
public Selection next() throws IOException {
    ...
    Proxy proxy = nextProxy();
    ...
    return new Selection(routes);
}

private Proxy nextProxy() throws IOException {
    ...
    resetNextInetSocketAddress(result);
    return result;
}

private void resetNextInetSocketAddress(Proxy proxy) throws IOException {
   ...
   List<InetAddress> addresses = address.dns().lookup(socketHost);
   ...
}
```

到这里差不多是网络较差的情况，也就是无法正常连接网络，就会抛出异常，一般来说dns是系统实现，看不了。而抛出异常就会打断拦截链导致后续的Interceptor无法执行，这也证明了NetworkInterceptor为啥只有网络正常时才会执行，当然还有各种情况，比如刚刚我们分析的，用了缓存，那么肯定是不会执行的。

## 总结
我们分析了一遍OkHttp的缓存实现，由于其遵守HTTP协议，我就不画流程图了，具体实现的理解可以跟着我的思路自己再看一遍源码，另外，OkHttp的源码可不止这些，即使全部看懂了，也没什么用，最重要的还是创意。真的很佩服这些大牛。


