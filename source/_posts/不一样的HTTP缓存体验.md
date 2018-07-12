---
title: 不一样的HTTP缓存体验
toc: true
date: 2018-06-24 17:43:11
tags: [Http, 缓存]
---

## 前言
继上篇[《来一份Android动画全家桶》](https://juejin.im/post/5b002a96f265da0b873ac44a)发布后，我相信你对Android的动画有一定的认识。这次我们讲解的内容是关于HTTP缓存，通过本篇我们不单单只是了解HTTP缓存机制，更重要的是学以致用，至于怎么用，嘿嘿。

![](/img/wotmnew.jpg)

> [温馨提示]对HTTP缓存已经有一定了解且对OkHttp缓存源码实现感兴趣，可以看看我写的[玩一玩OkHttp缓存源码](http://crazysunj.com/2018/06/24/%E7%8E%A9%E4%B8%80%E7%8E%A9OkHttp%E7%BC%93%E5%AD%98%E6%BA%90%E7%A0%81/)。

<!--  more-->
## HTTP缓存
我们试着自己实现一套HTTP缓存机制。首先我们必须了解HTTP是客户端请求服务器响应的标准。

### 客户端缓存
OK，假设我现在是服务器，有一个客户端请求我，我把他想要的内容响应给他，很愉快的一次交流。

好景不长，暴露出各种问题，例如有客户端反应我响应太慢，你自己网速差，距离我远，怪我喽？这都是还好的，可气的是每天不停地请求，甚至有时候同时多人请求，请求的内容还是重复的，我又不能不给他，我只想说住院费报销吗？

自己太累不要一个人扛着，说出来，于是我向老大反馈这件事情，老大不愧是老大，第二天就想出了一套方案。老大的方案我一遍就懂了(手动滑稽)，我这里给大家说说。

> 当客户端第一次访问服务器的时候，服务器如实把内容响应，其次在响应首部添加Expires，其值是一个GMT格式时间，告知客户端把内容在本地存一份，只要不超过这个时间，客户端请求时都直接读取本地文件。

我心里想着，如果客户端请求的内容具有时效性，那要是缓存，我们的名声岂不是一败涂地？作为一名优秀的搬砖工得提醒老大啊，老大听后露出欣慰的笑容。

> 如果想告知客户端不要缓存，那么服务器会在响应首部添加Pragma，其值是no-cache。

这是我们最初的版本(HTTP1.0)。但随着版本发布，出现了一个问题，我们无法保证客户端和服务器的时间一致，因为Expires的值是一个绝对时间，依赖于计算机时钟的正确设置。于是老大想出了用相对时间，哇，老大的形象在我心里又高大了。

> 服务器原先返回Expires的时候，另外添加Cache-Control，其值为max-age=相对时间值，单位是秒。Expires仍然可用(主要用于兼容)，优先级是Pragma -> Cache-Control -> Expires。

这是我们第二个版本(HTTP1.1)。XXX年后，客户端这帮家伙组团来到我们总部，声讨：你要我们缓存就缓存，不缓存就不缓存，我们不要的面子的啊？

![](/img/neixinwubodong.jpg)

行行行，你们来说。

> 客户端可以在请求首部添加Cache-Control，若其值为no-cache，那么不使用缓存而直接向服务器发出请求，但返回的不一定不是缓存，这是客户端期望的缓存策略。

### 服务器缓存
这群家伙自从有了这个规定，又让我们回到了过去，全给我no-cache，搞得我老大暴跳如雷。我知道，这时候是我的showtime，晋升指日可待。我告知老大，您当初的规范真的是一个伟大的决定，但可以用在客户端为何不能用在服务器呢？我老大深思一下，又欣慰对我一笑。

> 如果客户端缓存过期或者请求首部Cache-Control值为no-cache，会略过客户端缓存而直接向服务器请求，此时服务器采用条件方法再验证。

> 条件方法再验证一般使用两种条件首部：If-Modified-Since和If-None-Match。前者需要配合Last-Modified[其值是GMT时间，其意是文件的最后修改时间]，过程是客户端请求到服务器最新资源时，服务器会返回Last-Modified，当客户端再次请求服务器时，便会带上If-Modified-Since[其值是上次服务器返回的Last-Modified]，服务器会根据文件的最终修改时间与此比较，若一致，则返回304 Not Modified响应报文，反之，正常返回200。

大致过程如下：

![](/img/If-Modified-Since.png)

老大听完，先是对我的方案大赞一番，然后说可以改进，比如这两个场景：一个文件任你千万次修改，但内容不变；一个文件内容虽然改变了，但并不重要。哇！我对老大的敬佩之情犹如滔滔江水连绵不绝。

> 服务器返回ETag[其值一般是文件的hash]，时机与Last-Modified类似。客户端下次请求服务器时便带上If-None-Match[其值是Etag]，服务器会与之匹配，若匹配上，则返回304 Not Modified响应报文，反之，正常返回200。针对内容微改不影响主体，HTTP1.1支持"弱验证器"，即原先ETag添加前缀"W/"。

大致过程如下：

![](/img/If-None-Match.png)

高，实在是高！一股饭香扑鼻而来，低语，如果两者同时存在咋整？

![](/img/zhegediao.jpg)

老大说容我三思，然后出去了一趟，回来跟我说，这个简单。

> [RFC2616](https://www.rfc-editor.org/rfc/pdfrfc/rfc2616.txt.pdf)提到除非所有请求首部一致，不然不可返回304。后来RFC2616拆分成6份，其中[RFC7232](https://www.rfc-editor.org/rfc/pdfrfc/rfc7232.txt.pdf)提到如果两者同时存在，那么服务器可以自由发挥，可以两者都判断，也可以有优先级等等。

优秀啊！其实我还有一个问题，因为ETag会用一种算法去计算值，如果服务器采用了分布式(例如CDN)，会导致ETag不一致。其实算法保持一致就行啦。
### 缓存分类
真是一刻都不得清闲，客户端这帮家伙又来闹事，哭诉说，用户表示有些内容极其私密，只能偷偷看；用户表示经常访问的需要快速打开。

![](/img/tongqingxiao.jpg)

> 一般来说，缓存可以分为私有缓存和公有缓存。

#### 私有缓存
> 服务器返回Cache-Control，其值带有private，客户端将文件保存在本地并允许用户配置缓存信息，而服务器并不会缓存。

#### 公有缓存
> 服务器返回Cache-Control，其值带有public，默认为public，代理服务器就会把文件保存下来，客户端再次请求，若保存的文件可用，那么直接返回给客户端。代理服务器又被称为代理缓存。

XXX日后发布，从此世界和平！

第一次以故事形式讲解知识点，首先我来说句公道话，我觉得写得非常好，情节环环相扣又错综复杂(手动滑稽)！但你以为到这里就已经结束了吗？

![](/img/naive.jpg)
### 缓存层次结构
我们先前讲的客户端和服务器缓存明显的层次结构。首先客户端发出请求，先验证缓存是否过期，若未过期则使用(缓存命中)，此为一级缓存；若过期(缓存未命中)，那么继续向服务器请求，服务器验证(新鲜度检测)该缓存仍然可用则使用(再验证命中)，此为二级缓存；若不可用(再验证未命中)，那么服务器返回原始文件(200)，此为三级缓存；如果源服务器的文件已经被删除，那么返回404。

但理想很丰满，现实很骨感。例如分布式(CDN)，即使是较为易懂的单中心节点结构和多中心节点结构都比我们所说的链式结构复杂的多，更不用说网状结构(网状缓存)。其难点也不难发现，例如如何让缓存更快地更新或废弃，如何更快代价更低地让客户端获取缓存。

如果下一级缓存有多个选择，那么这些选择组成的缓存美其名曰兄弟缓存。

这里我们献上美图简单介绍本节内容：

![](/img/cdn_structure.png)

### 总结
这里我们针对上面所说做一个小总结。其实在上一节缓存层次结构已经跟大家过了一遍流程。我们的口号是什么？No picture，say a J8！

![](/img/http_cache_flow.png)

大佬发现有问题，望指正，感激不尽！
## 实战
理论终究只是理论，我们还是要回到日常！由于本人是个地地道道的Android搬砖工，所以你懂的。代码讲解基于Retrofit+OkHttp+HTTP1.1。

根据我们上面的分析，客户端首次请求的时候，服务端会返回一个Cache-Control响应首部来控制缓存。当然，后面我们也了解到客户端其实可以发起一个Cache-Control请求首部来期望自己的缓存策略。

```
@Headers("Cache-Control: public, max-age=300")//缓存时间为5分钟
@GET("random/data/{type}/{count}")
Flowable<GankioEntity> getGankio(@Path("type") String type, @Path("count") int count);
```

但最终的缓存策略还是由服务器控制，假设服务器并没有缓存策略呢？懵逼了吧？放心，困难总是没有办法多，OkHttp里面有个好玩的东西叫Interceptor，我们可以在这里到服务器返回的信息并修改。

```
private static class CrazyDailyCacheNetworkInterceptor implements Interceptor {
	...
    @Override
    public Response intercept(Chain chain) throws IOException {
        final Request request = chain.request();
        final Response response = chain.proceed(request);
        final String requestHeader = request.header(CACHE_CONTROL);
        //判断条件最好加上TextUtils.isEmpty(response.header(CACHE_CONTROL))来判断服务器是否返回缓存策略，如果返回，就按服务器的来，我这里全部客户端控制了
        if (!TextUtils.isEmpty(requestHeader)) {
            ...
            return response.newBuilder().header(CACHE_CONTROL, requestHeader).removeHeader("Pragma").build();
        }
        return response;
    }
}
```

首先我们取到Request，很明显，这是客户端的请求信息，然后再拿到服务器的信息Response，调用header修改Cache-Control的值，这里记得调用removeHeader("Pragma")，为何？还记得我们分析的Pragma的优先级是最高的吗？既然比不过他，那么将他移除我们就第一了。

![](/img/shehui.jpg)

那么如何告诉OkHttp呢？Android同学肯定很应手。

```
OkHttpClient.Builder builder = new OkHttpClient.Builder();
...
builder.addNetworkInterceptor(new CrazyDailyCacheNetworkInterceptor());
```

但是我们常常有在山洞的场景，那肯定没网啊！为啥常常在山洞？这个我们暂且放一边，没网肯定不可能到服务器啊，那我们也接受不到缓存策略。能不能在无网的时候，我们客户端制定一个缓存策略，比如无网时缓存支持一天，若超过一天，那么error。

```
private static class CrazyDailyCacheInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        CacheControl cacheControl = request.cacheControl();
        //header可控制不走这个逻辑
        boolean noCache = cacheControl.noCache() || cacheControl.noStore() || cacheControl.maxAgeSeconds() == 0;
        if (!noCache && !NetworkUtils.isNetworkAvailable()) {
            Request.Builder builder = request.newBuilder();
            ...
            CacheControl newCacheControl = new CacheControl.Builder().maxStale(1, TimeUnit.DAYS).build();
            request = builder.cacheControl(newCacheControl).build();
            return chain.proceed(request);
        }
        return chain.proceed(request);
    }
}

builder.addInterceptor(new CrazyDailyCacheInterceptor());
```

有了上面的分析，我相信代码并不难理解，这里有个注意点就是判断有无网的时候最好用ping的方法去检测，但这玩意是阻塞的，要注意。

那么，问题又来了，addInterceptor和addNetworkInterceptor有什么区别呢？一图胜千言，不多BB(官方)。而想知道具体原因，可以看我的[玩一玩OkHttp缓存源码](http://crazysunj.com/2018/06/24/%E7%8E%A9%E4%B8%80%E7%8E%A9OkHttp%E7%BC%93%E5%AD%98%E6%BA%90%E7%A0%81/)的扩展章节。

![](/img/okhttp_interceptor.png)

诶，是不是漏了什么？我们缓存放在哪儿呢？

```
//设置缓存 20M
Cache cache = new Cache(new File(context.getExternalCacheDir(), CacheConstant.CACHE_DIR_API), 20 * 1024 * 1024);
builder.cache(cache);
```

Android中的缓存实现就这么简单，简单？这是不可能的，这辈子都不可能，设计一套好的缓存策略是个大考验。

如果对OkHttp缓存源码实现感兴趣，可以看看我写的[玩一玩OkHttp缓存源码](http://crazysunj.com/2018/06/24/%E7%8E%A9%E4%B8%80%E7%8E%A9OkHttp%E7%BC%93%E5%AD%98%E6%BA%90%E7%A0%81/)。

## 骚聊
又到了紧张刺激的骚聊环节。HTTP缓存并不是什么新鲜的技术，但它却很重要，虽然我并没有总结HTTP缓存到底有什么好处，但并不难发现，例如它减少了带宽，减少了服务器压力，提高了用户体验等等。我们不要拘泥简单了解机制，而应该学习它的思想运用到我们开发中，例如我们老生常谈的图片三级缓存。人生就是痛并快乐着，加油吧，骚年！

最后，感谢一直支持我的人！
## 传送门
Github：[https://github.com/crazysunj/](https://github.com/crazysunj/)
