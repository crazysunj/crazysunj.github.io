---
title: 一起来玩Flutter
toc: true
date: 2019-01-22 18:29:33
tags: [Flutter,Dart]
---
## 前言
Flutter正式版1.0发布距今已经有一段时间，期间Flutter文章层出不穷，我会偷偷瞄几眼，最近实在是忍不住了，想自己玩玩，只能看难受得很（手动滑稽）。在此分享一下这几天对Flutter的学习心得，希望对新手村的同学有所帮助。

![](/img/ic_xixi_heihei.jpg)
<!--  more-->
## Flutter
### 简介
Flutter是Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过Dart语言开发App，一套代码同时运行在 iOS和Android平台。

这么专业的话我肯定说不出来，来自于[Flutter中文网](https://flutterchina.club/)。我依稀记得，我发表的[《一起来玩Weex》](https://juejin.im/post/5aae65546fb9a028cf324192)前言有提到混合开发的框架，但这次我觉得Flutter属于真正的跨端，譬如RN和WEEX等皆有“翻译”的过程。

为什么这么说呢？首先你可以认为Flutter是一套强大的UI框架，作为UI框架拥有自己的渲染引擎Skia，那么这就与平台无关，而Android系统已内置Skia；其次Flutter是基于Dart语言，典型的跨端采用统一语言，不用担心学习成本，跟Kotlin一样，如果你深入Java也可以很快入手Dart。毕竟Google是中意Java的。
### Android集成
确切来说是集成在已有app，对于大多数app来讲都是维护很久的工程，不太可能从头再来，那可是劳民伤财呀！不过官方可是Google，这会想不到？官方给出了[集成已有app方案](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps#experiment-turn-the-flutter-project-into-a-module)。

我这里简单介绍一下，顺便小提示下遇到的坑，毕竟从头开始简单，想要插入难呀！

![](/img/ic_huaji_daoli.jpg)

首先你得安装Flutter的开发环境，这还要我说吗？如果这都玩不好，抓紧转行，相信我，还来得及。

如果你已经安装成功，那你就可以用flutter命令啦。
#### 创建flutter工程
```
// 最好切到app工程根目录
flutter create -t module flutter的工程名
```
![](/img/ic_flutter_module.png)

执行命令后，会自动生成如上工程。这时flutter的插件工程已经生成，需要与主工程关联。
#### 关联flutter
```
// 在settings.gradle添加
setBinding(new Binding([gradle: this]))
evaluate(new File(
        settingsDir.parentFile,
        '/..include_flutter.groovy文件的相对路径../include_flutter.groovy'
))
```

同步一下，就会生成上方名为Flutter的library库。这里注意一下，自动生成的Flutter库是在.android目录下的，我特地copy出来，其一我采用的是AndroidX，自动生成的是support；其二自动生成的东西是会变的，别辛辛苦苦写的代码突然没了。copy的时候记得把include_flutter.groovy文件copy出来，因为这是连接flutter的关键，其实这些都可以改的，我们先老老实实不动。

![](/img/ic_guaiqiao.jpg)

#### 使用flutter
接下来，在我们的主module依赖Flutter即可。第一步的自动生成会给我们提供3个类:

* GeneratedPluginRegistrant：用于插件的统一注册，默认会在Lifecycle的onCreate中调用
* Flutter：关键类，用于生产FlutterView供原生使用
* FlutterFragment：返回FlutterView的Fragment的包装类

我们知道Flutter类可以帮我们生产FlutterView，那么我们很容易地把FlutterView添加到我们的原生容器中：

```
FlutterView gankioFlutterView = Flutter.createView(this, getLifecycle(), "CrazyDailyGankioFlutter");
...
mContainer.addView(gankioFlutterView);
```

原生是如何跟我们的dart组件关联起来的呢？关键的是第三个参数，官方采用的是理由机制，正如我们上面看到的第三个参数即是路由的url。

![](/img/ic_flutter_dart.png)

这里是编写flutter组件的地方。

```
void main() => runApp(crazyDailyWidgetRouter(window.defaultRouteName));

Widget crazyDailyWidgetRouter(String url) {
  switch (url) {
    case 'CrazyDailyGankioFlutter':
      return GankioFragment();
    ...
  }
}

class GankioFragment extends StatefulWidget {
  ...
}
```
现在我们可以尽情地写我们的flutter组件啦！
#### flutter组件
Flutter中组件称为Widget，一般分为有状态/[StatefulWidget](https://docs.flutter.io/flutter/widgets/StatefulWidget-class.html)和无状态/[StatelessWidget](https://docs.flutter.io/flutter/widgets/StatelessWidget-class.html)。

* 无状态：这个好理解，比如像[Text](https://docs.flutter.io/flutter/widgets/Text-class.html)这种，我传入什么文本就显示什么文本，不会动态改变，重写build方法，构建自己的布局就行了。

* 有状态：这个也不难理解，例如我组件的渲染内容会动态改变，比如请求了网络，根据网络数据来渲染，使用起来稍微啰嗦一点，需要创建一个[State](https://docs.flutter.io/flutter/widgets/State-class.html)。这个State管理着整个Widget的生命周期，常见的有initState，你可以认为是onCreate，其次是dispose，可以认为是onDestroy，记得要重写super方法哦。

```
@override
Widget build(BuildContext context) {
    return new ListView.builder(
      controller: mScrollController,
      padding: new EdgeInsets.fromLTRB(0, 5, 0, 5),
      itemBuilder: (BuildContext context, int index) => new GestureDetector(
            key: new Key(mGankioList[index].id),
            child: new Padding(
              ...
            ),
            onTap: () {
              ...
            },
          ),
      ...
    );
  }
```
这是gankio列表页的布局，相对比较具代表性就提出来讲解啦。就这种xml层级加dart代码格式层层嵌套，我估计能劝退不少人，哈哈。

从上面布局片段可知：

* 设置事件需要[GestureDetector](https://docs.flutter.io/flutter/widgets/GestureDetector-class.html)包装，有常用的[onTap](https://docs.flutter.io/flutter/cupertino/CupertinoTabBar/onTap.html)/单击、[onDoubleTap](https://docs.flutter.io/flutter/gestures/DoubleTapGestureRecognizer/onDoubleTap.html)/双击、[onLongPress](https://docs.flutter.io/flutter/gestures/LongPressGestureRecognizer/onLongPress.html)/长按等
* flutter也有id的概念，但平时基本不用，比如同级子Widget较多的情况可以采用，可以提高性能。原理有点类似React，采用组件树，根据[Key](https://docs.flutter.io/flutter/foundation/Key-class.html)为唯一识别符进行diff算法渲染树
* 屏蔽了margin的概念，如果需要在外层添加[Padding](https://docs.flutter.io/flutter/widgets/Padding-class.html)组件
* 事件监听，一般外层组件都会提供一个Controller，例如[ListView](https://docs.flutter.io/flutter/widgets/ListView-class.html)会提供一个[ScrollController](https://docs.flutter.io/flutter/widgets/ScrollController-class.html)

Flutter的组件到这就介绍的差不多了，想了解更多的官方组件可以点击[这里](https://flutterchina.club/widgets/)。这种都是慢慢来的，想想原生的一个TextView都能玩出各种各样的花样，路漫漫其修远兮。

![](/img/ic_liangliang.jpg)
#### 网络请求
网络请求的讨论似乎哪里都逃不掉，想想http的知识，突然一哆嗦。来看看Flutter的世界里是怎么样的吧！

```
get() async {
  // 创建 client
  var httpClient = new HttpClient();
  // 构造 Uri
  var uri = new Uri.http(
      'example.com', '/path1/path2', {'param1': '42', 'param2': 'foo'});
  // 发起请求, 等待请求，同时您也可以配置请求headers、 body
  var request = await httpClient.getUrl(uri);
  // 关闭请求, 等待响应
  var response = await request.close();
  // 解码响应的内容
  var responseBody = await response.transform(UTF8.decoder).join();
}
```
我表示这段代码没试过，出了事别找我，哈哈。我用的是中文网大佬们开源的[dio](https://github.com/flutterchina/dio)，这么叼，为什么不支持一下？

![](/img/ic_huaixiao.jpg)

```
void getGankioList() async {
  final String url = "...";
  List<ResultsEntity> list;
  try {
    final dio = new Dio();
    Response<Map<String, dynamic>> response = await dio.get(url);
    list = GankioEntity.fromJson(response.data).results;
  } catch (e) {
    ...
  }

  setState(() {
    ...
  });
}
```
看上去长短差不多，但我们怎么能以长短论英雄呢？多么痛的领悟啊！

细心的同学发现setState方法，这里我并没有注释，这个方法调用也很频繁，它到底什么时候调呢？其实它等价于runOnUiThread，自己体会吧！

更加细心的同学要问了，既然dio是第三方库，那么怎么依赖呢？原生我们可以靠gradle，Flutter里面我们可以靠pubspec.yaml，该文件在我们自动生成的文件的目录下。比如像这样，就可以依赖炫酷的dio库拉。

```
dependencies:
  ...
  dio: 1.0.13
  ...
```

#### JSON和序列化
有网络总是逃不掉json，这里的json好像不太友好，一般可这样用：

```
class User {
  final String name;
  final String email;

  User(this.name, this.email);

  User.fromJson(Map<String, dynamic> json)
      : name = json['name'],
        email = json['email'];

  Map<String, dynamic> toJson() =>
    {
      'name': name,
      'email': email,
    };
}
Map userMap = JSON.decode(json);
var user = new User.fromJson(userMap);
```
是不是很麻烦，每个实体类都得新增一个构造函数fromJson和方法toJson。Google深知麻烦，提供了json_serializable，具体用法我这里不讲解了，感兴趣的可以看[这里](https://flutterchina.club/json/)。我项目里的json解析暂时用的这种方法。

万事俱备，只欠运行。这时你兴高采烈地按下运行，卧槽，挂了；放屁，正常运行。这里Google貌似给我们挖了坑，可能部分同学会出现这个异常：

```
Must be able to initialize the VM
```
这时候你就得机灵点，肯定是sdk里面的东西出毛病了呀！说是容易，我也研究了长达2小时，各种与纯Flutter对比，发现我主工程并不会产生flutter_assets，但是我单独编译是会产生的，那么说明一个问题，插件有毒。稍微搜一下就发现flutter.gradle文件里这句话有毒：

```
Task mergeAssets = project.tasks.findByPath("app:merge${variant.name.capitalize()}Assets")
```
我想聪明的你已经知道了，一般两种改法，其一是重新定义主工程的名字替代app，你也可以提供一个变量，由外部去控制；其二是自己写一套copy逻辑。说放屁的应该是主工程名没改过，用的还是初始的app。

![](/img/ic_ruci_pishi.jpg)
#### 与原生交互
只要是混合开发，没有这个话题算我输！Flutter提供3种交互方式。

* [EventChannel](https://docs.flutter.io/flutter/services/EventChannel-class.html)：主要用于原生向Flutter端发送消息
* [MethodChannel](https://docs.flutter.io/flutter/services/MethodChannel-class.html)：主要用于Flutter端向原生发送消息
* [BasicMessageChannel](https://docs.flutter.io/flutter/services/BasicMessageChannel-class.html)：可以互传

三者底层均由[BinaryMessenger](https://docs.flutter.io/flutter/services/BinaryMessages-class.html)实现。
##### MethodChannel
这可能是我们用的较多的，也很符合我们的思维习惯，直接调用原生的方法。

```
// flutter端
final mGankioMethod = const MethodChannel('CrazyDaily/flutterGankioMethod');

// 这里的功能是通知原生SwipeRefreshLayout刷新结束
final Map<String, dynamic> params = <String, dynamic>{'type': type};
    await mGankioMethod.invokeMethod('refreshComplete', params);

// 原生端
// pluginRegistry由FlutterView获取
final MethodChannel channel = new MethodChannel(pluginRegistry.registrarFor(CHANNEL_NAME).messenger(), CHANNEL_NAME);
channel.setMethodCallHandler(new MethodChannel.MethodCallHandler() {
    @Override
    public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
        //  methodCall可以获取我们的方法名和参数，例如这里我们只要判断方法名是不是refreshComplete就能处理对应事件
        //  result是通知flutter端，支持异步回调
    }
});
```
##### EventChannel
我仔细想想这种场景相对是比较少的，一般更多的是flutter端发来一个通知，原生端再主动回复flutter端。像EventChannel这种，多用于系统级别监听，比如电量监听，这种持续的，谁用轮询谁知道。

```
// 原生端
final EventChannel channel = new EventChannel(pluginRegistry.registrarFor(channelName).messenger(), channelName);
channel.setStreamHandler(new EventChannel.StreamHandler() {
    @Override
    public void onListen(Object o, EventChannel.EventSink eventSink) {
        // 当flutter注册监听的时候，这里便会回调，一般我们会记录这个EventSink，因为后续需要通过它来通信
        mEventSink = eventSink;
    }

    @Override
    public void onCancel(Object o) {
        // 如果flutter端取消监听，这里便会回调
    }
});
// 通信方式很简单，除了success还有error和endOfStream，见名知意
private void onRefreshWithEvent() {
    if (mEventSink != null) {
        mEventSink.success(type);
    }
}

// flutter端
final mRefreshEvent = const EventChannel('CrazyDaily/flutterRefresh/Gankio');

// 不用感到奇怪，dart万物皆对象，函数也是一个对象，是可以当参数传递的
// 这个时候多了一种通信方式，就是把方法传入，子组件便可以通信父组件了
mRefreshSubscription ??
        mRefreshEvent.receiveBroadcastStream("refresh").listen(onRefreshEvent);

/// 通知子组件刷新
void onRefreshEvent(Object event) {
    ...
}

@override
void dispose() {
  super.dispose();
  // 记得注销
  mRefreshSubscription?.cancel();
  ...
}
```
##### BasicMessageChannel
正如前一节说的，我觉得这种场景是最多的，我们更希望每一次通信，都有一个结果回调，能够使整个过程更严谨，即使这个结果我不用，你也得给我，万一我又想用了呢？

![](/img/wotmnew.jpg)

```
// 原生端
mMessageChannel = new BasicMessageChannel<>(
        pluginRegistry.registrarFor("CrazyDaily/flutterGankioMessage").messenger(),
        "CrazyDaily/flutterGankioMessage",
        StandardMessageCodec.INSTANCE);

// 首参是发送的消息，一般为字符串
// 二参是消息回调，支持异步
mMessageChannel.send("...", new BasicMessageChannel.Reply<Object>() {
    @Override
    public void reply(Object o) {
        // o为回调信息
    }
});

// flutter端
final mGankioMessage = const BasicMessageChannel<Object>(
      'CrazyDaily/flutterGankioMessage', StandardMessageCodec());

mGankioMessage.setMessageHandler((message) async {
  // message便是我们原生端传的消息
  // 返回的信息便在原生端的reply方法接收
  return ...;
});
```
使用起来还算简单吧，可以适当的封装，这里只介绍原生端主动发送消息并接收结果，反过程一模一样，就不再多费篇幅了。
### 原理
如果写文章不是为了装逼，那将毫无意义。现在我就带你们一起飞！

![](/img/ic_flutter_architecture.png)

上图是Flutter的结构图，看看就行了，可以背出来，可能面试官会问，哈哈！

![](/img/ic_flutter_architecture_two.jpg)

上图是Flutter的渲染流程：

* 收到垂直同步信号，开始渲染
* Dart层输出图层
* 开始合成图层，可能会中断重来
* Skia引擎将图层处理成GPU数据
* 通过GL或者说Vulkan交给GPU开始渲染

![](/img/ic_flutter_architecture_three.jpg)

上图组件树渲染的一个过程：

* 动画触发Widget的state的改变
* state的改变触发Widget的重构
* 根据Diff计算出差异Widget并更新其布局
* 记录display list
* 输出图层，即第二张图中间部分

![](/img/ic_flutter_architecture_five.jpg)

上图是关于State的生命周期，但不仅限于此，可以点击[这里](https://book.flutterchina.club/chapter3/flutter_widget_intro.html)，列出了详细的生命周期变化。

## 骚聊
关于Flutter玩到这也差不多了，我不保证现在你已经熟悉甚至熟练Flutter，或者说理解Flutter，但是立即撸一个Flutter的Demo，简直So Easy！请注意你们青菜扔的位置，要是砸到花花草草就不好了。放心，虽然是空闲时间学习的，但我是认真的。很多东西官方都有，学着去发现，学习Flutter也不是看几篇文章就懂的，都是慢慢理解深入（滑稽）。有些同学肯定急了，文中代码有吗？在哪啊？看过我文章的都知道，大多数讲解内容都可在开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)找到。好了，差不多又要说再见的时候，嘿嘿，我还会回来的！

小弟在这里给各位大佬拜个早年！

![](/img/ic_xckl.jpg)

## 传送门
Github：[https://github.com/crazysunj/](https://github.com/crazysunj/)

博客：[http://crazysunj.com/](http://crazysunj.com/)

Flutter中文网：[https://flutterchina.club/](https://flutterchina.club/)

Flutter GitHub：[https://github.com/flutter/flutter](https://github.com/flutter/flutter)
