---
title: 一起来玩Weex
toc: true
date: 2018-03-16 17:20:57
tags: [Android,Weex,Js,Vue]
---

## 前言
从前几天的面试情况来看，绝大部分的公司提到的更多是Hybrid App，诸如ReactNative、weex和WebView这样的字眼。为啥呢？简单的总结了一下，发布节奏快（节约成本），跨端，前两者还可以拥有Native一样的流畅体验。那么到底是不是这样呢？我这种渣渣说了不算，时间会说明一切，而这篇文章主要讲的三者之一的weex。

在学习一门技术之前必须要有一定的觉悟，而我的觉悟就是：我现在没心情听他们什么狗屁weex现在和未来如何，我现在只想搞weex。

![](/img/huitouhuaji.gif)

## 效果图
这是我们这篇文章要撸的最后效果图，下面不仅会介绍如何撸，还会告知撸的时候遇到的问题和一些注意点。之所以选这样的场景是较具代表性。

![](/img/demo_weex.gif)

<!--more-->
## 实战
### 起因
老是看见大佬用weex开发了什么项目，光羡慕他们有什么用？我们要自己行动起来，比他们更厉害（手动滑稽）。

作为一名地地道道的Android工程师，在面试的时候问我rn或者weex的时候，真不知道咋说，主要我真不会啊，h5还好说，因为我在上家公司主要从事原生与h5的开发，同事也分享了许多前端的框架，但我基本了解下其中的思想，除了大学学的微不足道的js之外，真的是对前端一无所知啊！说了这么多跟主题无关的话，主要想说，不要害怕自己啥都不会，撸起袖子就是干。多一门技术多一份饭碗，提升自己的技术栈，更多的公司需要的是啥都会的人才（嘿嘿）。

	Tips:一些坑点和注意点会在讲述的时候掺杂，并不会一次性总结，大家要认真点哦！

![](/img/guaiqiao.gif)
### 开战
#### 基本布局
现在我们开始，关于什么环境配置，如何在Android studio上成功跑动demo类似这种我就不说了，你要时刻记住你是一名优秀的程序员，这篇文章要讲的更多的是Android视角，如何在已有项目中集成weex。

我们简单分析一下效果图，如果是我们原生开发，那么TabLayout+ViewPager+Fragment+RecyclerView是一种常见的选择。前三者在weex中并没有现成的组件，但RecyclerView有啊！在weex中有一个叫[list](https://weex.apache.org/cn/references/components/list.html)的组件可以达到这样的效果。先上最终的代码：

```
<template>
    <list class="list" @loadmore="loadmore" loadmoreoffset="10">
        <cell class="cell" v-for="(item,index) in lists" @click="itemclick(item,index)">
            <div class="panel1">
                <div class="panel2">
                    <text class="text1" lines="1">{{item.desc}}</text>
                    <text class="text2" lines="1">{{"作者："+isWhoEmpty(item.who)}}</text>
                    <text class="text3" lines="1">{{"发布时间："+getLocalTime(item.publishedAt)}}</text>
                </div>
                <image class="icon" src="mipmap://ic_go.png"></image>
            </div>
        </cell>

        <loading class="loading" :display="loadinging ? 'show' : 'hide'">
              <loading-indicator class="indicator"></loading-indicator>
              <text class="indicator-text">正在加载中...</text>
            </loading>
    </list>
</template>
```
关于下面的[loading](https://weex.apache.org/cn/references/components/loading.html)控件主要功能就是提供上拉加载，很简单，与之相配的是[refresh](https://weex.apache.org/cn/references/components/refresh.html)组件，聪明的你肯定已经猜到就是我们的下拉刷新。

list组件里面的loadmore是监听我们的上拉加载事件，相当于我们的Listener和下面[cell](https://weex.apache.org/cn/references/components/cell.html)的click都是监听事件。除此之外还有scroll事件，我们这里并没有用到。写法都是一样，前面写个@。

list与我们的RecyclerView一样只不过是容器，更好管理我们的条目，与我们item(ViewHolder)相对应的就是cell组件，这里你要知道一个思想，数据驱动UI，我们的数据源去控制我们UI的渲染。而weex是基于MVVM的，这里的[v-for](https://cn.vuejs.org/v2/api/#v-for)主要对集合的操作，item就是所处索引为index的值。相当于我们在onBindViewHolder根据position拿到数据，这很好理解。而cell组件里面的就是我们的onCreateViewHolder创建的view。

其中[text](https://weex.apache.org/cn/references/components/text.html)和[image](https://weex.apache.org/cn/references/components/image.html)我就不过多介绍了，相应链接都给出了。如果没接触过前端的同学肯定很好奇它是如何布局摆放的，像什么宽高啊，padding和margin啊，这其实是我们的[css](https://weex.apache.org/cn/wiki/common-styles.html)去控制的，但是是阉割版的，如果已经看过v-for链接的同学，发现是vue的官网，是的，weex基于vue，原来编写这样的文件格式为we，而现在要改为vue格式才能[编译](https://weex.apache.org/cn/tools/toolkit.html)成正确运行的js文件。

上面说了weex是MVVM模式，其实是vue采用MVVM模式，如果对Android的MVVM模式开发很熟悉的话，越看上面的文件越像我们的xml，我一直想说这个，xml在我们Android Studio中可以预览，如果上面的文件可以预览那该多好？No problem!我们可以利用Playground进行在线编写以及运行调试，甚至可以利用Playground的客户端扫码进行在手机端上调试，简直吊炸天。

#### Component
Component，也就是我上面说的组件，也就是我们的控件，除了官方内置的几种组件外，我们还可以自定义控件哦，是不是有点小兴奋，想当年也是因为自定义控件入行了Android，最前面所说的TabLayout+ViewPager+Fragment我把集成为一个控件，这样的做法，很明显的一个缺点就是耦合性增强了。我们现在处于学习嘛，怎么简单怎么来，先上我们的代码。

```
public class WXTabPagerComponent extends WXVContainer<LinearLayout>

@Override
protected LinearLayout initComponentHostView(@NonNull Context context) {
    LinearLayout root = (LinearLayout) View.inflate(context, R.layout.layout_tab_pager, null);
    ...
    return root;
}

@WXComponentProp(name = "titledata")
public void setData(List<String> datas) {
    if (mTabPagerAdapter == null) {
        ...
        mViewPager.setAdapter(mTabPagerAdapter);
        return;
    }
    mTabPagerAdapter.reload(datas);
}

<LinearLayout ...>
    <android.support.design.widget.TabLayout .../>
    <android.support.v4.view.ViewPager .../>
</LinearLayout>

WXSDKEngine.registerComponent("tabPager", WXTabPagerComponent.class);

<template>
    <div>
        <tabPager :titledata="datas" style="flex:1;"></tabPager>
    </div>
</template>
```
大部分无关紧要的代码都省略了，我相信你都可以脑补出来。最上面是我们是我们的自定义组件WXTabPagerComponent继承WXVContainer，还可以继承WXComponent，两者相当于我们的ViewGroup和View。

一般我们会重写initComponentHostView方法，也就是返回我们的自定义控件，我理解为它的泛型就是我们的自定义控件，return之后就是我们的component。

下面的setData方法这里可以理解为数据绑定方法，更准确的理解是属性方法，什么叫属性方法？我们看到最下面的vue代码，titledata就是我们setData的注解name，而datas就是一个字符串集合，上面我已经说过weex是基于MVVM的，如何进行绑定的下面再说。而如何解析成List<String>用的就是very牛皮的fastjson。所以setData这种属性方法你可以简单的理解为TextView的setText方法。

再下面是自定义控件的view层级。再下面是你必须注册该组件，不然weex不认识这玩意。

style="flex:1;"用来指定它的宽高，你可以理解为LinearLayout的weight属性，若不指定宽高，默认是0。

#### 数据绑定
接上面的自定义组件，贴数据绑定代码：

```
<script>
    export default{
        data () {
            return {
                datas: ["Android","iOS","前端"]
            }
        }
    }
</script>
```

这里的花样也是百花齐放，我学的时候也是一头雾水，比如带return属性写法表示只能在该组件生效，反之全局生效，可能会造成变量污染。这里的datas你可以理解我们的成员变量，就我们WXTabPagerComponent组件来说，我们return的datas直接以:titledata="datas"这样的形式进行绑定，一旦datas变量变化相应绑定的组件随之变化，所以叫属性方法。这里再多嘴一下，:titledata其实是缩写，全写是[v-bind](https://cn.vuejs.org/v2/api/#v-bind):titledata，这样是不是更好理解了？与最上面事件的@写法如出一辙，@写法的全写是v-on:，这些都是vue的东西。

![](/img/6fanle .jpg)

#### Native和JS通信
##### 自定义事件通知
该用法我还没用过，不做解释，我把官方的贴出。

```
public void fireEvent(String elementRef,final String type, final Map<String, Object> data,final Map<String, Object> domChanges){  }

public void fireEvent(String elementRef,final String type, final Map<String, Object> data){
  fireEvent(elementRef,type,data,null);
}

public void fireEvent(String elementRef, String type){
  fireEvent(ref,type,new HashMap<String, Object>());
}
```
具体解释可以戳[这里](https://weex.apache.org/cn/references/android-apis.html)。这种方法用于native主动调js。

##### 事件回调
这种方法用于js调native。

一种是带回调的，诸如：

```
public class WXLocation extends WXModule {
  @JSMethod
  public void getLocation(JSCallback callback){
    //获取定位代码.....
    Map<String,String> data=new HashMap<>();
    data.put("x","x");
    data.put("y","y");
    //通知一次
    callback.invoke(data);
    //持续通知
    callback.invokeAndKeepAlive(data);

    //invoke方法和invokeAndKeepAlive两个方法二选一
  }
}
```

一种是不带回调的，诸如：

```
public class RouterModule extends WXModule {
    @JSMethod(uiThread = true)
    public void router(String url) {
        BrowserActivity.start(mWXSDKInstance.getContext(), url);
    }
}
```
上面就是一个简单的路由，调用方法要用注解JSMethod，后面是确定是否在主线程调用。

双方使用也很简单：

```
WXSDKEngine.registerModule("RouterModule", RouterModule.class);//跟自定义组件一样，必须在原生注册

weex.requireModule('RouterModule').router(item.url);
```

#### Android扩展
确切来说应该是Adapter扩展，Adapter顾名思义就是适配器，就是说js某些地方可能需要原生来实现达到性能最优。

常见有：

* IWXUserTrackAdapter：相关性能数据 (首屏加载时间、JS-Native 通信时间、dom 更新时间等) 和其他通用信息 (JSLib 文件大小, Weex SDK 版本号等)，好像可以用于埋点统计
* IWXDebugAdapter：从字面上就能确定这玩意拿来调试的
* IWXStorageAdapter：不多说，用来存储的
* IWXImgLoaderAdapter：用来图片加载的，默认并没有实现，下面会给出自己实现的代码
* IWXHttpAdapter：用来支持网络下载的，默认使用HttpURLConnection，下面也会给出实现代码

```
public class WXImageAdapter implements IWXImgLoaderAdapter {
    @Override
    public void setImage(String url, ImageView view, WXImageQuality quality, WXImageStrategy strategy) {

        if (TextUtils.isEmpty(url)) {
            return;
        }
        final Context context = view.getContext();

        if (url.startsWith("mipmap://")) {//加载本地资源
            ...
            view.setImageResource(imgId);
            return;
        }

        Glide.with(context)
                .load(url)
                .crossFade()
                .into(view);
    }

    private String getResIdStr(String url) {
        ...
        return url.substring(start, end);
    }
}
```

```
public class WXHttpAdapter implements IWXHttpAdapter {

    private OkHttpClient mOkHttpClient;

    public WXHttpAdapter(OkHttpClient okHttpClient) {
        mOkHttpClient = okHttpClient;
    }

    @Override
    public void sendRequest(WXRequest request, OnHttpListener listener) {

        if (listener != null) {
            listener.onHttpStart();
        }

        if (request == null) {
            if (listener != null) {
                WXResponse wxResponse = new WXResponse();
                wxResponse.errorMsg = "WXRequest为空";
                listener.onHttpFinish(wxResponse);
            }
            return;
        }

        Request okHttpRequest;
        if (Constant.POST.equalsIgnoreCase(request.method)) {
            okHttpRequest = (new Request.Builder())
                    .headers(getHeaders(request))
                    .url(request.url)
                    .post(RequestBody.create(MediaType.parse(request.body), request.body))
                    .build();
        } else {
            okHttpRequest = (new okhttp3.Request.Builder()).url(request.url).build();
        }

        mOkHttpClient.newCall(okHttpRequest)
                .enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        if (listener != null) {
                            WXResponse wxResponse = new WXResponse();
                            wxResponse.errorCode = String.valueOf(-100);
                            wxResponse.statusCode = String.valueOf(-100);
                            wxResponse.errorMsg = e.getMessage();
                            try {
                                listener.onHttpFinish(wxResponse);
                            } catch (Exception e1) {
                                LoggerUtil.d(e1.getMessage());
                            }
                        }
                    }

                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        if (listener != null) {
                            WXResponse wxResponse = new WXResponse();
                            wxResponse.statusCode = String.valueOf(response.code());
                            ResponseBody body = response.body();
                            if (body == null) {
                                wxResponse.errorMsg = "body为空";
                            } else {
                                wxResponse.originalData = body.bytes();
                            }
                            try {
                                listener.onHttpFinish(wxResponse);
                            } catch (Exception e) {
                                LoggerUtil.d(e.getMessage());
                            }
                        }
                    }
                });
    }

    @TargetApi(24)
    private Headers getHeaders(WXRequest request) {
        okhttp3.Headers.Builder builder = new okhttp3.Headers.Builder();
        if (request.paramMap != null) {
            Set<String> keySet = request.paramMap.keySet();
            keySet.forEach(key -> builder.add(key, request.paramMap.get(key)));
        }
        return builder.build();
    }
}
```

代码篇幅比较长，但逻辑很简单，根据提供的参数WXRequest实现自己的网络请求，然后通过OnHttpListener回调给JS，其中注意点是尽量把回调给JS的变量WXResponse参数传齐，不然可能导致网络请求失败。

![](/img/zhixi.jpg)

#### 其它
我猜很多哥们内心现在最想问的是Android原生到底如何执行js文件？Weex的原理到底是什么？你那个效果图有bug，滑动的时候TabLayout透明了，你行不行啊？

![](/img/maolengjing.gif)

听我一一道来，关于Weex的[工作原理](https://weex.apache.org/cn/wiki/index.html)的示意图如下：

![](/img/weex_yuanli.png)

对于这种原理我也是半知半解，无能为力了。

关于Android原生执行js文件一般有两种，一种是从网络上获取，一种直接执行本地的js文件。

```
mWXSDKInstance.render("GankioList", WXFileUtils.loadAsset("gankiolist.js", context), options, null, WXRenderStrategy.APPEND_ASYNC);
mWXSDKInstance.renderByUrl();
```
前者是执行本地，后者是网络获取，我们常常会操作options以此给js传递一些native的参数。

关于最后一个问题，我也很无奈啊，原来这个项目就是卡片阴影背景，一开始发现weex也有这样的样式，欣喜若狂，后来发现这个问题，我一度以为我使用姿势错了，急忙回去看文档，天空飘来仅仅支持ios几个字。

![](/img/wotm.jpg)

为何ios这么叼？为何ios这么叼？为何ios这么叼？

## 杂谈
这算是对自己这几天学习Weex的一点小总结，小弟不才，已经尽力了，希望对想入门或者刚入门Weex的同学有所帮助，同时希望Weex的前辈能够指出其中错误。文中的代码已经集成到我开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)。学习Weex主要还是看重它基于Vue，传说Vue是以为长得很帅的国人创立的，请给我一个理由不支持？

![](/img/zhuangbiciji.gif)

最后，感谢一直支持我的人！
## 传送门
Github：[https://github.com/crazysunj/](https://github.com/crazysunj/)

博客：[http://crazysunj.com/](http://crazysunj.com/)

Weex常用网站：[https://github.com/joggerplus/awesome-weex](https://github.com/joggerplus/awesome-weex)