---
title: 玩一玩Android下载框架
toc: true
date: 2018-08-12 20:39:10
tags: [下载框架]
---
## 前言
继上篇[《不一样的HTTP缓存体验》](https://juejin.im/post/5b2fb529e51d4558864932e6)已经有一段时间了，一直没写教学型文章不是因为太忙，想了很久不知道以什么为主题，有个哥们看了我的开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)，好像对下载挺感兴趣，那我就写一篇吧！下载框架似乎是我们入门必学的一个技术点，因为它囊括了很多方面的知识，优秀的开源下载框架非常多，各有千秋。那么，此刻，大家一起跟着我来打造一款下载框架！准备好了吗？

![](/img/kaishibiaoyan_small.jpg)

## 效果
一贯作风！No picture，say a J8！

![](/img/download_effect.gif)
<!--more-->

## 实战
我们从效果图上简单分析一下执行流程，首先打开二维码，扫描一个下载链接；解析到下载链接跳转下载中转页并弹出下载信息确认框；确认下载，通知栏回显进度；下载完成，点击可查看。

OK，很实用的一个流程，扫码和下载可以算是我们天天都会用到的两个技术。
### 扫码
没什么悬念，我选择zxing，谷歌出品，必属精品。这里我选择zxing-android-embedded，它是基于zxing简单封装，可扩展，用不用其实无所谓，我们的重点并不在这里。我们看到的二维码效果，本身并不是这样的，我写的效果是模仿微信的，那是如何做的呢？文末告诉你答案。

zxing的使用很简单，这样就调起扫码界面了。记得请求摄像头权限哦。

```
new IntentIntegrator(this)
        .setCaptureActivity(ScannerActivity.class).initiateScan();
```
ScannerActivity是我们自定义的扫码界面，支持从本地图片中扫码，实现过程忽略。扫完码肯定会回调一串字符串，例如这里我们是一个下载链接，那么在哪里回调呢？

```
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    IntentResult result = IntentIntegrator.parseActivityResult(resultCode, data);
    String scanResult = result.getContents();
    if (scanResult != null) {
        BrowserActivity.start(this, scanResult);
    }
}
```
回调结果谷歌都给你封装好了，你直接拿来用就是，这里我们需要那串字符串。大家应该知道了，我们的中转页其实是个Web页。

![](/img/heiren_wtf.jpg)

### 中转页
那么为什么要用Web页当中转页呢？其实感觉用中转页可能不太合适，扫码结果页可能更合适一点。我们扫码结果多种多样，常见的是一个http链接，其又分为网页和下载链接。如果人工去判断实在太麻烦了，即使只有http的。我们可以简单把字符串分为两类，一类是符合URI格式的，另一类是不符合的。如果不符合URI格式，那么我们直接把它当成文本处理；反之，我们直接用WebView去解析。

前面已经说了，即使是只有http的也是很麻烦的，你如何判断一个链接是下载链接呢？靠后缀名吗？不存在的，唯一的办法就是连接之后解析。实在太麻烦了，如果玩过WebView的同学肯定知道WebView支持下载监听的，浏览器内核会帮我们去解析，我们只要实现这一的监听就结束了。

```
setDownloadListener(new DownloadListener() {
    @Override
    public void onDownloadStart(String url, String userAgent, String contentDisposition, String mimeType, long contentLength) {
        if (mDownloadCallback != null) {
            mDownloadCallback.onDownload(url, contentLength);
        }
    }
});
```

OK，回调的肯定在我们的中转页：

```
mWebView.setDownloadCallback((url, contentLength) ->
        new AlertDialog.Builder(this, R.style.NormalDialog)
                .setTitle("提示")
                .setCancelable(false)
                .setMessage(String.format("下载链接：%s\n下载大小：%sMB", url, StorageUtil.byteToMB(contentLength)))
                .setNegativeButton("不下", null)
                .setPositiveButton("下载", (dialogInterface, i) -> DownloadService.start(this, url))
                .show());
```

简单的一个弹框，确认下载跳转我们的下载服务。

但你以为这真的会弹出来来吗？

![](/img/neixinwubodong.jpg)

测试中有些手机并不会弹出(并没有回调DownloadListener)，但如果是正常网页可以加载出来且点击网页中的下载链接也可以下载，如果是这样就好办了，其实只要在页面加载前，再加载一次就完事了。例如这样：

```
@Override
public void onPageStarted(WebView webView, String s, Bitmap bitmap) {
    if (!isLoaded) {
        isLoaded = true;
        webView.loadUrl(s);
    }
    super.onPageStarted(webView, s, bitmap);
}
```
记得用isLoaded去控制哦，不然是个闭环，仔细想想，哈哈。

### 核心下载
关于下载，我们这里设计成启动服务在后台下载。
#### 网络请求
```
mPresenter.download(url, FileUtil.getDownloadFile(this)); // 启动下载
```
Presenter连接我们的Model层，

```
mDownloadUseCase.execute(DownloadUseCase.Params.get(url, saveFile), new BaseSubscriber<File>() {
    @Override
    public void onNext(File file) {
        mView.onSuccess(file);
    }

    @Override
    public void onError(Throwable e) {
        super.onError(e);
        mView.onFailed(e);
    }

    @Override
    public void onComplete() {
        mView.onComplete();
    }
});
```
domain层调用我们的data获取下载数据，

```
@Override
protected Flowable<File> buildUseCaseObservable(Params params) {
    return mDownloadRepository.download(params.url, params.saveFileDir);
}

@Override
public Flowable<File> download(String url, File saveFileDir) {
    return mDownloadService.download(url)
            .observeOn(Schedulers.io())
            .map(response -> convertFile(saveFileDir, response))
            .subscribeOn(Schedulers.io())
            .unsubscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread());
}
```

最终还是我们熟悉的Retrofit，哈哈。

```
@Streaming
@GET
Flowable<Response<ResponseBody>> download(@Url String url);
```
简单介绍下Streaming和Url注解，Streaming可以响应立即以字节流返回，默认会把数据全部加载到内存中，所以可以用于大文件下载；Url可以指定请求路径，覆盖本身的baseurl。

#### 回显进度
既然我们要回传进度，retrofit是如何回调进度的呢？准确点应该是okhttp，okhttp一个比较核心的东西叫Interceptor，我们可以通过这知道当前下载进度。

```
private static class ProgressInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Response originalResponse = chain.proceed(chain.request());
        return originalResponse.newBuilder()
                .body(new ProgressResponseBody(originalResponse.body()))
                .build();
    }
}
```
重新封装我们的Response，如果对拦截不太了解的可以看看我这篇文章[《玩一玩OkHttp缓存源码》](http://crazysunj.com/2018/06/24/%E7%8E%A9%E4%B8%80%E7%8E%A9OkHttp%E7%BC%93%E5%AD%98%E6%BA%90%E7%A0%81/)，然后重新封装body，

```
private static class ProgressResponseBody extends ResponseBody {
    ...
    public ProgressResponseBody(ResponseBody responseBody) {
        this.responseBody = responseBody;
    }
    ...
    @Override
    public BufferedSource source() {
        if (bufferedSource == null) {
            bufferedSource = Okio.buffer(source(contentLength(), responseBody.source()));
        }
        return bufferedSource;
    }

    private Source source(long contentLength, Source source) {
        return new ForwardingSource(source) {
            long bytesReaded = 0;
            @Override
            public long read(Buffer sink, long byteCount) throws IOException {
                long bytesRead = super.read(sink, byteCount);
                bytesReaded += bytesRead == -1 ? 0 : bytesRead;
                RxBus.getDefault().post(String.valueOf(taskId), new DownloadEvent(contentLength, bytesReaded));
                return bytesRead;
            }
        };
    }
}
```

source表示输入流，不懂的可以看看我这篇[《玩一玩Okio源码》](http://crazysunj.com/2018/07/08/%E7%8E%A9%E4%B8%80%E7%8E%A9Okio%E6%BA%90%E7%A0%81/)，当初分析okhttp源码的时候，有人不太理解，故后面补了这么一篇来分析okio的源码。如果已经了解的同学，肯定就懂了，bytesRead就是我们每次读入的字节，我们创建成员变量bytesReaded支持每次回调就加上bytesRead来统计当前已经读入的总字节，contentLength方法就是整个需要读入的总字节。而这里我们通过RxBus来把这个数据分发出去。RxBus底层其实就是RxJava，感兴趣自己去看看，这里不多介绍。

但这其实是有问题的，敏锐的同学已经发现了，但先不说问题，我们继续后面的步骤。

刚刚谈到RxBus，既然有发布方，那么必须要有个订阅方，在我们的DownloadPresenter中，

```
mDownloadUseCase.execute(RxBus.getDefault().toFlowable(tag, DownloadEvent.class), new DisposableSubscriber<DownloadEvent>() {
    @Override
    public void onNext(DownloadEvent downloadEvent) {
        final int progress = (int) (downloadEvent.loaded * 100f / downloadEvent.total + 0.5f);
        mView.onProgress(progress);
    }
    ...
});
```

OK，很简单就是回调给我们的View层。再来看看我们的View层怎么写的。

```
@Override
public void onProgress(int progress) {
    mNotificationBuilder.setContentText(String.format(Locale.getDefault(), "正在下载:%d%%", progress))
            .setProgress(100, progress, false);
    mNotificationManager.notify(NOTIFICATION_ID, mNotificationBuilder.build());
}
```

#### 通知栏
不停改变通知栏的进度，那么通知栏如何创建呢？

```
private void initNotification() {
    mNotificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        // 适配通知栏8.0
        assert mNotificationManager != null;
        // 需要创建通知渠道，比如我们这里的通知栏是用于下载监听的
        NotificationChannel channel = mNotificationManager.getNotificationChannel(CHANNEL_ID_DOWNLOAD);
        if (channel == null) {
            channel = new NotificationChannel(CHANNEL_ID_DOWNLOAD, "下载通知", NotificationManager.IMPORTANCE_MIN);
            mNotificationManager.createNotificationChannel(channel);
        }
        if (channel.getImportance() == NotificationManager.IMPORTANCE_NONE) {
            // 通知栏权限没开启，可直接跳转到权限设置界面
            Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
            intent.putExtra(Settings.EXTRA_APP_PACKAGE, getPackageName());
            intent.putExtra(Settings.EXTRA_CHANNEL_ID, channel.getId());
            startActivity(intent);
            Toast.makeText(this, "设置好通知栏权限，请重新下载", Toast.LENGTH_SHORT).show();
            stopSelf();
        }
    }
    // 初始化下载通知栏
    mNotificationBuilder = new NotificationCompat.Builder(this, CHANNEL_ID_DOWNLOAD)
            .setContentText("正在下载")
            .setSmallIcon(R.mipmap.ic_launcher)
            .setOngoing(true)
            .setWhen(System.currentTimeMillis());
    mNotificationManager.notify(NOTIFICATION_ID, mNotificationBuilder.build());
    Toast.makeText(this, "正在下载，可在通知栏查看进度哦", Toast.LENGTH_SHORT).show();
}
```

该注释的我都注释了，下载进度我们已经处理好了，那么下载完，我们如何处理呢？我们肯定是要将文件保存在本地，然后通知栏告知下载完成，点击查看。
#### 保存文件
因为保存文件是统一逻辑，所以写在data层，还记得DownloadDataRepository的download方法吗？再来一遍：

```
@Override
public Flowable<File> download(String url, File saveFileDir) {
    return mDownloadService.download(url)
            .observeOn(Schedulers.io())
            .map(response -> convertFile(saveFileDir, response))
            .subscribeOn(Schedulers.io())
            .unsubscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread());
}
```
显然convertFile方法就是保存文件的关键方法啦，大家一起来看看吧：

```
@Nullable
private File convertFile(File saveFileDir, Response<ResponseBody> response) {
    final ResponseBody responseBody = response.body();
    ...
    try {
        File saveFile = new File(saveFileDir, getFileName(response));
        bufferedSink = Okio.buffer(Okio.sink(saveFile));
        source = Okio.source(responseBody.byteStream());
        bufferedSink.writeAll(source);
        bufferedSink.flush();
        return saveFile;
    } catch (IOException e) {
       	...
    } finally {
        ...
    }
    return null;
}
```
熟悉okio的同学已经知道怎么回事了，不熟悉的也没关系，其实就是用okio写入我们要保存的文件里，没什么难点。这里的难点其实是如何得到保存文件名，当然最简单的就是用户自己去设置。但我们要的肯定是自己动啊。

![](/img/kaiche.png)

stop！我们来看看getFileName方法。

```
@NonNull
private String getFileName(Response<ResponseBody> response) {
    final okhttp3.Response raw = response.raw();
    // 得到contentDisposition
    String contentDisposition = raw.header("Content-Disposition");
    String fileName;
    if (TextUtils.isEmpty(contentDisposition)) {
        // 如果为空，那么我们就不能在这里取了，咋办？只能靠截取下载链接了，记得把参数截掉。	
        String file = raw.request().url().url().getFile();
        fileName = file.substring(file.lastIndexOf("/") + 1, file.contains("?") ? file.indexOf("?") : file.length());
    } else {
        // 如果存在，那么很简单了，filename后面的就是文件的名字
        try {
            fileName = URLDecoder.decode(contentDisposition.substring(contentDisposition.indexOf("filename=") + 9), "UTF-8");
            fileName = fileName.split("\"")[fileName.contains("\"") ? 1 : 0];
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
            fileName = contentDisposition.substring(contentDisposition.indexOf("filename=") + 9);
            fileName = fileName.split("\"")[fileName.contains("\"") ? 1 : 0];
        }
    }
    return fileName;
}
```

这里其实也有坑点，下面再说。data层处理完后，最终还是会回调到View层。

```
@Override
public void onSuccess(File saveFile) {
    Toast.makeText(this, "下载完成，保存路径：" + saveFile.getAbsolutePath(), Toast.LENGTH_SHORT).show();
    Intent intent = new Intent(Intent.ACTION_VIEW);
    intent.addCategory(Intent.CATEGORY_DEFAULT);
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    Uri uri;
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
        uri = FileProvider.getUriForFile(this, getString(R.string.file_provider_authorities), saveFile);
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    } else {
        uri = Uri.fromFile(saveFile);
    }
    intent.setData(uri);
    PendingIntent pendingintent = PendingIntent.getActivity(this, 0, intent, PendingIntent
            .FLAG_UPDATE_CURRENT);
    mNotificationBuilder.setContentIntent(pendingintent);
}
```
设置通知栏的跳转链接，url指向我们的保存文件，注意，这里我们要兼容7.0。现在都9.0了，这应该不需要我说了吧？不存在的，即使10.0，我估计我5.0还没搞清楚。

![](/img/genbushang.png)

## 扩展
一个简单的下载框架我们算是完成了，但还是存在多个小毛病，说是小毛病，其实是致命的，哈哈。
### 多任务
第一个我留下的问题是，用户多次扫码多次创建下载任务会如何？我们的RxBus是全局的且事件并没有标识任务，也就是说所有的任务都会在通知栏回调，那效果不堪入目啊。有的同学我猜会顺着这思路问，那下载任务也会创建多个吗？会不会只有一个啊？提问题很好，但基础貌似不过关，Service多次调用startService启动，那么onCreate只会执行一次，但onStartCommand会执行多次。

对于替换掉RxBus而使用listener我更倾向RxBus事件加个标识，标识当前的任务，这样做正好与通知栏的标识相对应。假设是这样，那么用什么来当标识？简单点就是用时间戳，但是我们的通知栏ID貌似是int类型，MMP。想了下，还是老办法，用UUID，大概率是不会重复的，那么重复了怎么办？再获取一次。那么，我们可以定下获取任务ID或者说通知栏ID的方法。

```
private int getTaskId() {
    do {
        int taskId = UUID.randomUUID().hashCode();
        if (mTaskIds.indexOfKey(taskId) == -1) {
            return taskId;
        }
    } while (true);
}
```

由于我们的通知栏标识跟我们的任务ID一致，故添加新的数据存储集合用来存储我们的通知栏实例。

```
private SparseArray<DownloadInfo> mTaskIds = new SparseArray<>();

private class DownloadInfo {
    NotificationCompat.Builder builder;
    boolean isComplete; // 用于全部下载完成，自动关闭service
    ...
}
```
那么接下来就很简单了，处理的时候，只要把原来的通知栏替换成集合根据taskId取到的实例即可。再则，必须把taskId一直传到我们的ProgressResponseBody中，然后：

```
RxBus.getDefault().post(String.valueOf(taskId), new DownloadEvent(taskId, contentLength, bytesReaded));
```
我们的事件DownloadEvent添加了新属性taskId。

### 文件名
接下来我们说说关于fileName的坑点，为什么会有坑点呢？我。。。为啥会问这样的问题。。。很简单啊，因为文件覆盖了呗，处理逻辑也很简单，如果当前正常存储文件名已经存在，那么重命名直至不存在，这样无论单任务还是多任务都会保存相应的文件，而不会覆盖。当然了，更友好一点，提示用户要不要重新下载啊？我们这里就直接再下一个，就是这么暴力。

那么这里的难点就是修改我们的fileName。最终代码：

```
@NonNull
private String getFileName(File saveFileDir, Response<ResponseBody> response) {
    ...
    if (TextUtils.isEmpty(contentDisposition)) {
        ...
    } else {
        ...
    }
    int count = 0;
    String temFileName = fileName;
    String fileNamePrefix;
    String fileNameSuffix;
    int pointIndex = fileName.lastIndexOf(".");
    if (pointIndex > -1) {
        fileNamePrefix = fileName.substring(0, pointIndex);
        fileNameSuffix = fileName.substring(pointIndex, fileName.length());
    } else {
        fileNamePrefix = fileName;
        fileNameSuffix = "";
    }
    do {
        File saveFile = new File(saveFileDir, temFileName);
        if (saveFile.exists()) {
            temFileName = String.format(Locale.getDefault(), "%s(%d)%s", fileNamePrefix, ++count, fileNameSuffix);
        } else {
            fileName = temFileName;
            break;
        }
    } while (true);
    return fileName;
}
```
稍微有点小操作，模仿了下谷歌浏览器的下载命名规则，哈哈，后面加个(1)这样子的，好像很多都是这样子的，这个也很好实现，首先把以前的文件名分为前缀和后缀，分隔符是"."。那么只要在"."前面添加()就OK啦，其次判断当前文件名是否存在，如果存在数字+1，否则就是我们最终的文件名。

那么问题来啦，除了这两个坑还有其它吗？有，肯定有。例如回调进度的时候不用每次都更新，可以隔一段时间或者说隔一定进度。

![](/img/wotm.jpg)

## 骚聊
整篇文章下来，发现实现一个下载框架也不过如此？是的，它真的不是很难，难点我觉得有两个，一个是下载框架的视觉交互，这个好像等于没说，哈哈；一个是兼容性，好像这个也等于没说；最后一个是高并发下载。逗我，你连最基础的断点续传都没讲。。。确实没讲，但这个难吗？当然了，自己并不是专门开发下载工具的，难点也只是自己猜的，勿怪。

还有一点就是很多人比较关心的，文中的代码在哪里可以看到？这么说吧，我发的教学型文章的代码90%来自自己的开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)，readme也列出了技术点，如果对哪一点比较感兴趣，可以看看！特别重要的一点是有问题一定要说出来，不要害羞，无论是谁的问题。

当然这次千万别问我为什么扫的是知乎！！！

大家下次再见！
## 传送门
Github：[https://github.com/crazysunj/](https://github.com/crazysunj/)

博客：[http://crazysunj.com/](http://crazysunj.com/)


