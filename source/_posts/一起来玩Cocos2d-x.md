---
title: 一起来玩Cocos2d-x
toc: true
date: 2019-05-25 09:22:11
tags: [C++,Cocos2d-x]
---
## 前言
首先，我先申明我是一名Android工程师，用到[Cocos2d-x](https://zh.wikipedia.org/wiki/Cocos2d)是工作需要，技术选型选择Cocos2d-x我想得益于它仅有的开源免费跨平台吧，其中跨平台还借力于OpenGL。如果你是一名游戏工程师不放去看看[Unity](https://zh.wikipedia.org/wiki/Unity_%28%E6%B8%B8%E6%88%8F%E5%BC%95%E6%93%8E%29)，自己比较一下，我这里就简单吐槽一下Cocos2d-x，毕竟Unity没用过。最先接触的吐槽点就是官方文档，算了，不说了，看在提供了中文文档的份上；然后就是UI系统，能体系完整点么？能文档里列全点吗？能使用人性化点吗？你们不考虑性能问题吗？还有一点就是版本的兼容，网上查资料api都变了，艹了个DJ，想查的内容根本找不到，不像我大Android，一查便知。这是我学了一周的感受。B话就讲到这，想玩一下Cocos2d-x的同学跟着我的节奏来吧！

![](/img/img_xinlei.jpg)

**注:**本文Cocos2d-x基于C++，版本3.17.1；NDK版本15；代码内容均是我开发的打飞机项目的一部分。

## Cocos2d-x
### 概念
Cocos2d-x是MIT许可证下发布的一款功能强大的开源游戏引擎。允许开发人员使用C++、Javascript及Lua三种语言来进行游戏开发。支持所有常见平台，包括 iOS、Android、Windows、macOS、Linux。
<!--  more-->
不用多想，官方文档复制，很牛皮。

![](/img/img_niu_pi_a.jpg)

### 核心
Cocos2d-x的核心东西大致有这么几点，导演、场景、层、节点、精灵、动作、粒子、物理引擎、菜单和事件分发。
#### 导演
导演对应的类是[Director](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/d7/df3/classcocos2d_1_1_director.html)，采用的是单例模式，主要用于管理场景和一些配置信息。一般来说，我们可以通过Director实现场景切换：

```
// 你可以理解场景就是我们的Activity
// 运行一个场景
auto scene = Shooting::createScene();
Director::getInstance()->runWithScene(scene);
// replaceScene是替换场景，除此之外还能切换到下一个或者返回上一个场景等
auto first = FirstScene::createScene();
auto trans = TransitionRotoZoom::create(2.0f, first);
Director::getInstance()->replaceScene(trans);
```

除此之外，还可以配置一些信息，如屏幕，这里牵涉到屏幕适配，后面会单独讲：

```
// 我们配置设计分辨率是1920*1080，其中glview是我们的GLSurfaceView，我们选择的适配策略是宽不变，高自适应，你可以理解成今日头条的适配方案，一个比例的问题
auto designSize = Size(1920, 1080);
glview->setDesignResolutionSize(designSize.width, designSize.height,
                                    ResolutionPolicy::FIXED_WIDTH);
// 获取我们设置的设计屏幕尺寸，除此之外可以获取实际屏幕尺寸和可见尺寸等
auto winSize = Director::getInstance()->getWinSize();
```

#### 场景
场景对应类是[Scene](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/d4/d5f/classcocos2d_1_1_scene.html)，我们看到各种各样画面就是场景，不多介绍了。

```
// 创建一个自释放的普通场景，运行及切换场景导演类已经介绍过了
auto scene = cocos2d::Scene::create();
// 创建真实的物理场景
auto scene = Scene::createWithPhysics();
```

#### 层
层对应的类是[Layer](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/df/de4/classcocos2d_1_1_layer.html)，我们平时写的更多的是各种各样的层，你可以理解层就是我们的fragment:

```
// 我们常常实现自己的层，然后添加到场景中
class Shooting : public Layer
// 以下是生命周期方法
// 记得重写init初始化方法，系统回调
// init回调的时候获取的设计尺寸是原始尺寸，注意，最好在onEnter之后初始化位置大小
bool init() override;
// 进入层时候调用
void onEnter() override;
// 进入层而且过渡动画结束时候调用
void onEnterTransitionDidFinish() override;
// 退出层时候调用
void onExit() override;
// 退出层而且开始过渡动画时候调用
void onExitTransitionDidStart() override;
// 层对象被清除时候调用
void cleanup() override;
// UI组件是没有背景的，如果想要纯色背景，可以这样
// Color4B是一个色值类，分别对应RGBA，当时没看清踩了个坑，我说为啥有点透明呢
auto bg = LayerColor::create(Color4B(110, 110, 110, 255));
// 创建完层，记得添加到场景中
auto layer = Shooting::create();
scene->addChild(layer);
```

#### 精灵
精灵对应类是[Sprite](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/d3/d5c/classcocos2d_1_1_sprite.html)，官方是这么介绍的：准确的说，精灵(Sprite)是一个能通过改变自身的属性：角度，位置，缩放，颜色等，变成可控制动画的2D图像。可能多了一点是精灵有自己的精灵缓存，提供了缓存机制。

```
// 普通的精灵创建
auto sprite_shooting = Sprite::create("Images/sprite_shooting.png");
// 以下两种均没用过，所以给我的感觉精灵跟UI组件没什么区别
// 把一个图集直接加载到精灵缓存中，plist文件中带有属性
auto spritecache = SpriteFrameCache::getInstance();
spritecache->addSpriteFramesWithFile("sprites.plist");
// 直接创建一个精灵并放到精灵缓存中
auto mysprite = Sprite::createWithSpriteFrameName("mysprite.png");
// 精灵添加到层中(在层Shooting类函数中执行)
this->addChild(sprite_shooting);
```

以上提到的导演、场景、层和精灵的大致关系如下(请叫我盗图狂魔)：

![](/img/cocos2d_guanxi.png)

#### 菜单
类似于Android的menu。

```
// 打飞机项目中没有用到菜单，贴一下最初学习时敲的
// 创建多个菜单切换项，除了MenuItemToggle还有MenuItemLabel和MenuItemSprite
auto font1 = MenuItemFont::create("10");
font1->setColor(Color3B::RED);
auto font2 = MenuItemFont::create("100");
auto font3 = MenuItemFont::create("1000");
// 设置菜单的点击事件，注意，真的只是点击事件，点一下就回调
// 像打飞机方向按键及导弹发射按键都是长按的，只能单击，这谁顶得住啊
// 可变参数记得传NULL
// CC_CALLBACK_1是cocos2d定义的回调宏，一共有4种
auto menuToggle = MenuItemToggle::createWithCallback(
        CC_CALLBACK_1(MyTest::menuItemCallback, this),
        font1,
        font2,
        font3, NULL);
// 创建菜单对象，添加对应的菜单节点
Menu *menu = Menu::create(menuToggle, NULL);
// 回调函数，可以根据指针信息判断对应菜单作逻辑处理
void MyTest::menuItemCallback(cocos2d::Ref *p);
// 添加到层中，第二个参数是层级，谁上谁下的问题
this->addChild(menu, 4);
```

#### 节点
节点对应类是[Node](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/d3/d82/classcocos2d_1_1_node.html)，从官方介绍，我们可以知道Node可以锁是非常基础的类，贯穿着大多数组件，比如前面提到的场景、层、精灵和菜单均继承Node(多级)。

由于它的基础性，这里列出它的一些常用函数:

```
// 前面有介绍到可以增加一个孩子到相应节点
scene->addChild(layer);
// 移除一个孩子，第一个参数是移除节点，第二个参数是是否清除，这里牵涉到cocos2d-x的内存管理，后面会讲
this->removeChild(bullet, true);
// 从父亲中移除并清除自身，例如我们弹出对话框后再关闭，对话框的原理我们后面介绍
removeFromParentAndCleanup(true);
// 可以给节点设置一个名字，并通过名字获取对应节点，同理还可以通过Tag操作
score->setName("score");
auto score = getChildByName("score");
// 节点作一个动效
bullet->runAction(anim);
// 启动一个定时器，第一个参数是回调函数，参数是对应帧，第二参数是间隔时间循环一次，当然了还有其它种类定时器，可以理解为我们的Handler
schedule(schedule_selector(Shooting::update), 5);
// 节点设置位置
menu_up->setPosition(Vec2(250, 400));
// 节点进行缩放，除此之外还有旋转、倾斜、锚点、内容大小、可见性等
menu_up->setScale(scale);
// 节点同样的也支持生命周期，除此之外的跟层介绍的一致
virtual void onEnter();
// 获取对应场景，有点我们获取Context类似
virtual Scene* getScene() const;
// 每个节点可以绑定一个数据结构或者说一个对象
virtual void setUserData(void *userData);
virtual void setUserObject(Ref *userObject);
```

#### 动作
cocos2d-x是一个游戏引擎，那么动作这块是非常重要的。我这里苦下决心给大伙整理一张关系图，请叫我雷锋！

![](/img/img_cocos_action.png)

使用起来一开始可能不太习惯，因为此套动作结构更强调的是组合，针对单个动作可能达不到你想要的效果。

```
// 执行一个动作
bullet->runAction(anim);
// 判断一个节点是否正在执行动作
child->isRunning()
// 停止当前节点所有动作
child->stopAllActions();
// 停止指定动作
void stopAction(Action* action);
// 组合动作，带回调
auto callback = CallFuncN::create(CC_CALLBACK_1(Shooting::actionEndCallback, this));
auto anim = Sequence::create(moveTo, callback, NULL);
```

上面的图其实可以慢慢看，用到的时候再来查！

![](/img/img_wa_bi_kong.jpg)

#### 粒子
what's the [lizi](https://zh.wikipedia.org/wiki/%E7%B2%92%E5%AD%90%E7%B3%BB%E7%BB%9F)?

![](/img/img_lizi.gif)

放这张图是方便理解，粒子大致就是一个个小点根据某种物理轨迹执行某种运动形成较为真实感的视觉效果。除了这种星辰效果外，常常用于模拟火、雨、雪、爆炸和流星等。

```
// 以下是实现导弹击毁敌人时，敌人爆炸的代码
// 创建爆炸粒子
auto explosion = ParticleExplosion::create();
// 设置执行时间周期
explosion->setDuration(0.5f);
explosion->setLife(0.5f);
// 太大了，缩放了一下 - -!
explosion->setScale(0.4);
// 粒子也要添加到当前层
this->addChild(explosion);
// 设置纹理图片
explosion->setTexture(Director::getInstance()->getTextureCache()->addImage("Images/stars.png"));
// 设置粒子动画结束自动移除并清除
explosion->setAutoRemoveOnFinish(true);
// 设置爆炸位置
explosion->setPosition(vec2);
```

#### 物理引擎
物理引擎顾名思义就是可以模仿真实世界的物理运动规律，例如自由落体、抛物线、碰撞和反弹等。

在cocos2d-x的世界里是怎么实现的呢？

首先需要的是一个物理世界，需要一个存在的空间：

```
// 创建真实的物理世界
auto scene = Scene::createWithPhysics();
```

在这世界里有各种各样的物体：

```
// 创建一个导弹
ImageView *bullet = ImageView::create("Images/icon_bullet.png");
...
// 创建一个物理世界的空壳套在我们的组件上
// 在这里你还可以设置空壳的形状，总不能就是一个矩形吧，如果是这样，我明明没碰到却算我碰到了，这...不过我懒得写了
// 除此之外你还可以添加关节，什么意思呢？比如飞机两侧可以添加额外的导弹组件，有点变形金刚的味道，这样就合为一体
auto body = PhysicsBody::createBox(bullet->getContentSize());
// 以下是检测碰撞代码
// 这里大有学问，举个例子
// body1->setCategoryBitmask(0x02) body1设置类别掩码 0010
// body1->setCollisionBitmask(0x01) body1设置碰撞掩码 0001
// body2->setCategoryBitmask(0x04) body2设置类别掩码 0100
// body2->setCollisionBitmask(0x06) body2设置碰撞掩码 0110
// 此时body1的碰撞掩码与body2的类别掩码作逻辑与运算值为0，说明body1不能与body2发生碰撞
// 相反body2的碰撞掩码与body1的类别掩码作逻辑与运算不为0，说明允许body2能与body1发生碰撞
// 接触检测原理同上
// 类别掩码
body->setCategoryBitmask(0x01);
// 碰撞掩码
body->setCollisionBitmask(0x02);
// 接触掩码
body->setContactTestBitmask(0x01);
// 带上套子
bullet->setPhysicsBody(body);
```

程序毕竟是程序，需要一个监听的过程，不像真实世界那么自然：

```
contactListener = EventListenerPhysicsContact::create();
contactListener->onContactBegin = [this](PhysicsContact &contact) {
    // 获取到碰撞时的两个节点
    Node *nodeA = contact.getShapeA()->getBody()->getNode();
    Node *nodeB = contact.getShapeB()->getBody()->getNode();
    int nodeATag = nodeA->getTag();
    int nodeBTag = nodeB->getTag();
    if (nodeATag == SpriteType::bullet && nodeBTag == SpriteType::enemy) {
        // SpriteType是枚举类，如果是导弹和敌人碰撞，执行相应的逻辑，当然了还有其它各种碰撞情况，代码省略了
        handleColliding(nodeA, nodeB);
    }
    ...
    return false;
};
// 导演去设置监听
Director::getInstance()->getEventDispatcher()->addEventListenerWithSceneGraphPriority(
        contactListener, this);
```

#### 事件分发
存在交互就逃不掉事件分发，cocos2d-x的事件分发具体分了3种角色：

* 事件
	* EventTouch：触摸事件
	* EventMouse：鼠标事件
	* EventKeyboard：键盘事件
	* EventAcceleration：加速度事件
	* EventCustom：自定义事件
	* PhysicsContact：物理碰撞事件，前面已经接触过了
* 事件源：各种节点
* 事件处理者
	* EventListenerTouchOneByOne：单点触摸事件
	* EventListenerTouchAllAtOnce：多点触摸事件
	* EventListenerKeyboard：键盘事件
	* EventListenerMouse：鼠标事件
	* EventListenerAcceleration：加速度事件
	* EventListenerController：游戏手柄事件
	* EventListenerFocus：焦点事件
	* EventListenerCustom：自定义事件
	* EventListenerTouchPhysicsContact：物理碰撞事件  

分发的事件，分发的事件监听，分发事件的监听地方都齐了，就缺一个分发的人了。通过前面物理引擎章节，我们已经知道分发器是什么了：

```
Director::getInstance()->getEventDispatcher();
// 分发一个事件，常用于自定义事件
void dispatchEvent(Event* event);
// 添加一个监听，传入监听事件和监听对象
void addEventListenerWithSceneGraphPriority(EventListener* listener, Node* node);
// 移除一个监听事件
void removeEventListener(EventListener* listener);
```

OK，这里介绍几个常见的事件监听单点触摸和键盘事件：

```
# EventListenerTouchOneByOne 单点触摸
typedef std::function<bool(Touch*, Event*)> ccTouchBeganCallback;
typedef std::function<void(Touch*, Event*)> ccTouchCallback;

// 触摸开始，同Android的DOWN事件
ccTouchBeganCallback onTouchBegan;
// 移动状态，同MOVE事件，这里有个坑，如果手指保持同一点位置将不会回调
ccTouchCallback onTouchMoved;
// 手抬起，同UP事件
ccTouchCallback onTouchEnded;
// 事件被取消，同CANCEL事件
ccTouchCallback onTouchCancelled;

# EventListenerKeyboard 键盘事件
// 按下时触发
std::function<void(EventKeyboard::KeyCode, Event*)> onKeyPressed;
// 按键被释放时触发
std::function<void(EventKeyboard::KeyCode, Event*)> onKeyReleased;
```

还有一点忘记介绍了，就是事件分发的路线，大致是这样的，我们每次add一个节点的时候，可以传入第二个参数z-order，后者数值越大越先被分配事件，同样的cocos2d-x中也有事件拦截一说，通过返回bool判断，同Android，被拦截的事件将不会被下级节点接受。

### UI系统
#### 系统组件
据说cocos2d时候是没有这么多GUI组件的，只能用用标签、菜单和精灵，据说能满足开发的需求。后面增加GUI组件是因为方便ide的各种拖拽，大哥你是不是搞错前后顺序了？

![](/img/img_shangtian.jpg)

好了，废话不多说，UI系统无非是大量的UI组件罗列，本节会整理常用的组件结构图和一些使用例子。

![](/img/img_cocos_ui.png)

上面是常用的组件结构图，画的可真累啊。如果你对Android的UI系统有所了解话，肯定是能明白这些组件的具体功能，可能在cocos2d-x中如何使用不太清楚，这里会列举项目中几个例子供大家参考。

```
// 创建失败后弹窗再来一把的按钮
auto btn = Button::create("Images/CyanSquare.png");
// 对按钮中添加文字
btn->setTitleText("再搞一把");
// 按钮可以添加触摸事件
btn->addTouchEventListener(CC_CALLBACK_2(NormalDialog::btnCallback, this));
// 创建得分文本，传入字符串，字体和字体大小
auto score = Text::create(text, "fonts/fangzheng.ttf", 60);
// 创建一个导弹图片
ImageView *bullet = ImageView::create("Images/icon_bullet.png");
```

#### 自定义组件
除了这些系统提供的之外，常常需要我们自定义组件，例如我们打游戏的时候经常会弹出对话框，但我们发现系统组件里面并没有很常见的Dialog，是不是很懵逼？因为cocos2d-x认为弹窗其实就是一个层，所以你自定义对话框样式的层之后添加到原来的层就行了。

#### 3D
cocos2d-x是支持3D的，但是如果要用3D我肯定不会用这个，就比如现在很多Android库功能是真TM全，但这样好吗？太笨重，很多功能我压根就不用，占包体积，很多方面都兼顾但做得都很烂等等原因，导致不得不修改源码把需要的提出来。同样的，你加入了3D的功能，OK，能不能单独提出来一个库？

#### 坐标系
cocos2d-x中坐标系有两种，我的天就不能统一一下吗？

其中默认坐标原点是在左下角的即OpenGL的坐标系。

![](/img/opengl_cs.png)

但有时我们会用到同Android坐标系即坐标原点在左上角即UI坐标。

```
// class/Touch
// 这样获取的便是左上角
Vec2 getLocationInView() const;
// class Director
// UI坐标转OpenGL坐标
Vec2 convertToGL(const Vec2& point);
// OpenGL坐标转UI坐标
Vec2 convertToUI(const Vec2& point);
```

#### 屏幕适配
只要是一个平台分辨率有多种，那必然存在适配问题。cocos2d-x中存在3种分辨率：

* 屏幕分辨率：没啥好解释的
* 资源分辨率：比如我们加载的资源图片，有兴趣的可以去研究下Android资源目录下的x,xx,xxx图片，即使同一张图片，系统加载的分辨率均不一样
* 设计分辨率：可以认为是我们编程时的分辨率，也就是我们给它适配的分辨率，标准UI的分辨率，不知道这样说能否理解

其中适配策略有5种：

* ResolutionPolicy::NO_BORDER：无边策略，计算设计分辨率和屏幕分辨率的缩放因子，保证其中一个维度可以正好铺满屏幕，另一个维度是超出屏幕外的
* ResolutionPolicy::FIXED_HEIGHT：NO_BORDER升级版，不是通过计算确定正好铺满的维度而是通过指定，该策略是指定高度，是的，就是炒的很火的今日头条方案，不知道几年前的东西了
* ResolutionPolicy::FIXED_WIDTH：同上，指定宽度
* ResolutionPolicy::FIXED_ALL：完全按设计分辨率来，上面的是填充一个维度之后，另一个维度是超出屏幕的，而这种策略是会留边的即会有黑边
* ResolutionPolicy::FIXED_FIT：宽的维度等比例缩放，高的维度等比例缩放，可以精准适配，就是百分比布局，但是有个致命的缺点就是图片会变形（压缩或者拉伸）

说了这么多，其实Android项目没有自己的适配方案的同学可以参考这个，总之，没有完美的适配，按照自己项目的实际情况进行适配。

#### 其它
其它例如音视频这种跟平台有关系的，比如在Android平台用的就是MediaPlayer，当然你可以改源码，毕竟开源嘛。音乐播放是通过[SimpleAudioEngine](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/de/d8f/class_cocos_denshion_1_1_simple_audio_engine.html)控制：

```
// 预加载一段音频
SimpleAudioEngine::getInstance()->preloadBackgroundMusic("audio/LuckyDay.mp3");
// 播放一段音乐，第二个参数是控制音乐循环
SimpleAudioEngine::getInstance()->playBackgroundMusic("audio/LuckyDay.mp3", false);
```

### 网络
#### 基本原理
像这种很明显是跟平台有关系的，cocos2d-x中添加了相应封装API。Android平台底层是HttpURLConnection，可以改相应源码或者覆盖，例如换成优秀的OkHttp。

```
// 创建一个请求类
HttpRequest *request = new HttpRequest();
// 传入请求url
request->setUrl(...);
// 设置请求类型
request->setRequestType(HttpRequest::Type::GET);
request->setTag("get");
// 设置网络请求回调
request->setResponseCallback(CC_CALLBACK_2(FirstScene::parseResponse, this));
// 发起请求
HttpClient::getInstance()->send(request);
// 记得释放请求
request->release();
```

#### JSON解析
cocos2d-x同样提供了JSON解析API，jsoncpp和rapidjson。这里举例后者的使用：

```
// 创建json字符串
const char *json = "{\"project\":\"rapidjson\",\"stars\":10}";
// 创建解析对象
Document d;
// 解析json字符串
d.Parse(json);
// 访问对应键值，值被封装成Value
rapidjson::Value &s = d["stars"];
// 从Value中获取对应值并打印
__android_log_print(ANDROID_LOG_ERROR, "MyTest", "json---%d", s.GetInt());
```

#### 数据持久化
牵涉到数据处理，这块永远逃不掉。cocos2d-x提供一套算凑活的方案，例如常见的文件操作可以通过[FileUtils](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/dc/d69/classcocos2d_1_1_file_utils.html)来解决常见文件创建、读和写等。除了一般的文件操作，cocos2d-x也提供了与Android的SharedPreferences类似类[UserDefault](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/db/d94/classcocos2d_1_1_user_default.html)，存储也是key-value的形式。最后cocos2d-x也支持数据库存储，不多说了，采用sqlite，但这并不是内置的，需要自己[官方](https://www.sqlite.org/download.html)额外下载解压。

### 内存管理
我想说的是，我开发的cocos2d-x是基于C++的，那么内存管理自然是C++的那一套，除此之外，cocos2d-x自己封装了一套便于开发者管理内存，毕竟C++那一套内存泄漏真的是防不胜防。

![](/img/img_wo_dong_ni.jpg)

如果对前面的内容看的比较仔细的同学会注意到cocos2d-x的类似乎都继承[Ref](https://docs.cocos2d-x.org/api-ref/cplusplus/V3.17/df/d28/classcocos2d_1_1_ref.html)，如果点进去看一下源码很容易发现cocos2d-x管理内存是通过[引用计数](https://zh.wikipedia.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)的方法。

cocos2d-x中的对象基本都是Node，而Node是通过addChild和removeChild，那么removeChild的时候对象会被delete掉。

那么如果不是Node呢？或者说我的Node未addChild且又不想被释放怎么办？提供了这几种方案：如果一个引用打算立即释放，那么可以通过调用release函数，底层就是delete掉；如果想跟Java这种自动管理内存的可以通过调用autorelease函数，Ref中会有PoolManager类来管理引用，如果一帧内不使用，就会被自动释放；如果想引用不因为不被引用了(计数为0)而被释放掉，可以调用retain函数，底层就是计数+1操作。

## 结束语
欢乐的时光总是短暂的，我现在的心情是如释重负，这么diao的游戏引擎框架被我轻描淡写总结完了。内容确实挺多的，还有许多开发技巧我也是一脸懵逼，但总是有这么一个过渡，刚开始，卧槽，这个简单，你看，跑起来了；哦，这个这样用啊，我敲一下，牛皮，可以的；写个小Demo，东拼西凑，完美；多个项目实战，卧槽，不对啊，按照实例上来的呀，怎么不行啊，卧槽，我Demo的时候也这么写的呀，咋又不行了啊...无限奔溃中。嗯，给自己吃了颗定心丸，毕竟我也才学1周不到嘛，况且人家这框架这么diao。言归正传，许多细节没有写清楚，这需要经验的积累，卧槽，我笔记本上为啥有头发？卧槽，腰有点痛？卧槽，头有点晕？

下...下期再...再见！

![](/img/img_zaijian_meizi.jpg)

![](/img/img_xingxing.jpg)
