---
title: MTRVA2.0来啦
toc: true
date: 2018-04-03 19:50:16
tags: [Android]
---
## 前言
从MTRVA的诞生到现在已经有一段时间了，当初第一个版本出来之后，我是很激动的，这可以说是我真正意义上的第一次创作，也是自己的一次历史性进步，写到这，又想起自己熬夜更新版本，躺在床上呆呆地思考着哪里需要优化修改...太多太多的回忆，那么MTRVA作为RecyclerViewAdapter的一股清流，到底有什么鸟用呢？
## 特点
* 使用简单快捷，可配合大多数Adapter
* 一行代码刷新单个level，可对应多个type，刷新带有动画
* 支持增删改查操作
* 支持异步，高频率刷新，可扩展(如配合RxJava)
* 单个level支持Loading(加载)，Empty(空)，Error(错误)页面切换
* 单个level支持header，footer
* 单个level支持展开和合拢(可设置合拢最小值)
* 支持加载全局Loading(加载)页面
* 支持注解生成类，减少工作量
* 支持刷新生命周期回调
* 兼容低版本RecyclerView
* 进阶用法，比如打造自己的headerView和footerView，让页面在多种页面之间自由切换

## 思想
![](/img/mtrva_architecture.png)

这是MTRVA的架构图。
<!--  more-->
我们先想象这样一个常见的场景（列表页），那么我们首先要写一个Adapter，然后针对我们每一个type写一个xml，其次我们要在Adapter中用一个list来控制item数量以及每个item渲染所需要的实体类等等，如果对原有list插入逻辑很复杂，甚至复杂到要控制type的start和end的position，可能最初写逻辑代码的时候你很快写完了，幸运的是也没报错。悲剧的是过了几个月，突然报错了，再回去看的时候，我相信你会和我一样，心情只能“卧槽”来形容。或者在一堆type控制position的代码中再加一个type，对不起，我选择辞职。

![](/img/gaoci.jpg)

当然，辞职是开玩笑的，MTRVA这时候就发挥作用啦，在这里，你不用担心数据逻辑处理（而且这些处理也不应该写在你的业务层代码中），你只要关心你的UI逻辑，而这些是你应该关心的，也必须关心的，也必须用心写的，因为我们的业务不一样，我们不一样。

那么机智的同学肯定要问，既然MTRVA是控制数据处理的，那是不是类似一个Helper类与Adapter组合使用，还是传统的改造Adapter的架构达到目的？这个问题水平很高，奖励你一个女朋友，MTRVA另一大亮点是与Adapter配合使用，只要你的Adapter是RecyclerViewAdapter，那么基本可以和MTRVA配合使用，如果有不能配合的，请联系我，我尽量不影响你项目Adapter的使用。

这里有个小插曲，有几个同学反映即使只写UI逻辑，如果type够多的话，代码依然很复杂，这其实不难，你可以定义一个处理UI逻辑的接口，针对于不同type实现接口，把不同type的UI逻辑给抽离出来，甚至可以复用。如果脑中还是没有一个清晰的模型，可以参考我的demo，这是我应该做的（手动滑稽），但不要照搬，适合自己的才是最好的。

## 实战
关于实战我是真的不想再写了，因为实在太简单了，这里给出我在掘金发的一篇文章[《BRVAH+MTRVA，复杂？不存在的》](https://juejin.im/post/59913263f265da3e331cde55)以及MTRVA的[文档](http://crazysunj.com/2017/08/14/MTRVA%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E%E4%B9%A6/)，看不懂的你打死我，而这次主要想写的是2.0更新了什么？如果我项目已经集成了MTRVA，我需要注意什么？我更想写的是这些，当初并不想发文章，可是我想着有些同学可能用了一个版本之后，如果不是那个版本已经支持不了需求，是不会去更新版本的，用的好好的我干嘛去更新啊？当这些同学想更新的时候可能只能去看文档，而文档只有最新的使用方式，因此我秉承一贯的作风，每次更新我都会写一篇文章来阐述版本之间的差距，而这次属于大版本更新，动静稍微大了点。

这里说个很多人问的问题，关于网格和线性混排的问题。这个其实不属于我的范畴，但我肯定会细心解答的，最简单的就是嵌套，如果优雅一点可以研究下类GridLayoutManager的setSpanSizeLookup方法，可以参考这篇文章[《RecyclerView:实现带header的grid》](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0722/3214.html)。

## 更新
既然是大版本更新，到底有什么骚操作呢？
### 背景
我们先从大方向来，这次改版的核心思想是从type的思维定势到level的重视和突破。这样说可能有点抽象，俗话说的好，No picture,say a J8!我们来看看2.0版本的布局模型图与1.0版本的比较。
![](/img/mtrva1.0.png)
![](/img/mtrva2.0.png)
如果是老用户会发现2.0版本的设计更符合我们的初衷。这次的改版跟大家意见和建议有着密不可分的关系，一个人能力再强也是有限的（这样夸自己真的好吗），而团队的力量是无限的，真的很感谢你们。

好了，我们简单分析一下，这为什么是突破。从1.0版本看，明明最外层定义的是level，而刷新的级别却是type；注册资源的时候，一个type去对应一个level，这是矫情吗，我要这level有何用？如果两种type同一level使用起来特别不方便，局限性较大。而我们再看2.0版本：

![](/img/niubimao.jpg)

一种level可以注册多种type，在level里面type数据的顺序是无限制的，例如上面的type2在type1的中间，显然是可以无视顺序，让type1堆在一起这种，这样我们的level可控制的UI更加丰富！虽然2.0版本之前也能实现，但对操作要求较高。

这是一次思想的突破，总是避免不了api的改动，而这时候就体现架构设计的重要性，你是一个一个去找出来再改还是改一个地方就行了，你是要删这删那，还是只要添加类就行了...这是我们的语言，而专业一点，你要牢记Java的六大设计原则，虽然我用的也不够灵活，但是你不去用就永远也不会用。

我猜有些同学已经不耐烦了，能不能说重点，到底改了啥啊？兄台，请放下你的40米大刀，我马上说。
### API差异
#### 资源注册
首先我们从注册资源开始，原来我们是这样的：

```
registerMoudle(ItemEntity3.TYPE_3)
                .level(2)
                .layoutResId(R.layout.item_3)
                .register();
```
而现在是这样的：

```
registerMoudle(LEVEL_FIRST)
                .type(TypeOneItem.TYPE_ONE)
                .layoutResId(R.layout.item_first)
                .type(TypeTwoItem.TYPE_TWO)
                .layoutResId(R.layout.item_second)
                .register();
```
与上面说的一样，以level为主导，下面我们可以定义多个type，每个type对应一个布局xml，原来level的定义更多指的是type顺序的等级，而其它诸如header、footer、loading等不变。

#### 刷新方法
其次一系列的notify方法，原先的type参数全部改为level，这也是影响比较大的地方，如果你是按照我的建议把helper刷新方法封装在adapter里面，改起来也是so easy!瞧，这就是设计的魅力！

例如常用的这些方法：

```
public void notifyMoudleDataChanged(List<? extends T> data, int level);
public void notifyMoudleDataAndHeaderChanged(List<? extends T> data, T header, int level);
public void notifyMoudleDataAndHeaderAndFooterChanged(T header, List<? extends T> data, T footer, int level);
```
#### 资源适配器
LoadingEntityAdapter修改点是一样的，原先回调参数只有type，这次多加一个参数level，虽然两者只要得其一就能计算出另一个。举个栗子：

```
T createLoadingEntity(int type);===>T createLoadingEntity(int type, int level);
```
#### 方法
* getLevel方法从private改为public
* 去除getCurrentRefreshType方法并用getCurrentRefreshLevel方法替换
* 全局初始化loading(加载)类LoadingConfig以及相应Builder的setLoading方法参数type改为level
* 新增getDataWithLevel方法，根据level拿到相对应的数据
* 类HandleBase的type属性修改为level，这也是意料之中的事情

#### 注解
我不知道这玩意有人用吗？反正先前版本已经分离出来了，如果想用要另外依赖库，具体可以看文档。

关于注解的改动其实和上面差不多，主要把原先定义为type的通通改为level，使用方法是不变的。

## 杂谈
就改动这么点东西？也要兴师动众吗？Sorry，就这么点东西，打扰了。甚至你可以无脑记type改level，但...但是我底层逻辑的修改可惨了，我已经记不清我改了多少地方，我只记得我一直在疯狂测试，生怕哪里没改动而影响升级了2.0版本的同学使用。当然已经集成MTRVA的同学并不一定要更新，这次是在架构上的一次改进，方便以后的扩展。如果跟我一样，喜欢新版本的同学可以尝试更新版本，这里提供自己常用的更新新版本方法：首先下载官方提供的demo，疯狂玩，玩demo不过瘾的自己改代码再玩，玩得差不多的时候就集成到项目中，疯狂自测，没问题之后与版本测试一起交给我们可爱的测试人员。如果有更好的方法可以联系我，我改完告知大家，大家一起学习。

细心的朋友发现我们的篇幅并不是很长，本次的主角就是MTRVA，就是一个库，但对我来说是一种情怀。我先说说我自己选择库的标准，首先确定库的特点是什么？对我现在的项目有什么好处？我现在项目中有没有类似的库，相比较怎么样？如果我选择使用，使用是否简单，集成是否复杂，会不会对我项目有很大影响，后期作者是否维护等等。这是自己的想法，希望能帮到你，欢迎补充。

本来这个月发的文章主题是Android性能优化，之前公司是小公司（虽然现在还是小公司），属于需求完成就万事大吉，但作为一名优秀的程序员，对自己开发的应用要求仅仅满足需求，那也太low了，迟早被淘汰。下个月可能发布我对性能优化的一点理解，希望对大家有所帮助，拭目以待！这里插个嘴，我的项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)也跟着更新了，除了MTRVA的更新，还有替换内涵段子为搞笑视频，原因你懂的！

这里强调一点，我开源的库我肯定会一直更新下去，除非没人用了或者我已经不在这行业了。虽然有时候多个用户在同一段时间问了好几个问题，有几个还是比较棘手的问题，自己也曾想过删库或者停止维护。但想想别人用我的库是对我的一种认可，是看得起我，同时满足自己微不足道的虚荣心，我瞬间就来劲了，哈哈，不知道这是不是自我催眠。以后开源库可能会少写，但开源的精神必须有，一些比较装逼且实用的写法我都会集成在我的项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)中，喜欢的朋友可以star关注一波，剧透一波，将有一大波自定义控件与动画涌入项目。

最最最重要的一点是，大家可以在我的[博客](http://crazysunj.com/)找到我的联系方式，随便加，但不一定即时回复，敬请谅解。通过好友后第一句不要叫我大佬（手动捂脸），我只是一只开发Android的渣渣。有几个同学问我为啥不组个群啊？我相信你肯定加了一堆的群，有没有用，你心里还没有一点13数吗？有问题就直接问好了！可能你的问题我也没碰到过，但我会尽力给你解决，大家一起学习，何乐而不为？当然能搜索到的就别来啦！

最后，感谢一直支持我的人！

等等，那...那个...帮忙...那个...点...点...点下star。

![](/img/kelianxixi.jpg)
## 传送门
Github：[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

文档：[http://crazysunj.com/2017/08/14/MTRVA%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E%E4%B9%A6/](http://crazysunj.com/2017/08/14/MTRVA%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E%E4%B9%A6/)

