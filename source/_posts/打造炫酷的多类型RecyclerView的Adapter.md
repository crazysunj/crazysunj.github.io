---
title: 打造炫酷的多类型RecyclerView的Adapter
date: 2017-04-01 14:10:12
tags: [Android,RecyclerView,Adapter,DiffUtil]
toc: false
---

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RecyclerView渐渐地走进了我们的项目里面，UI越来越复杂，高效快捷开发出复杂的页面是我们所追求的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;相信大家对BaseRecyclerViewAdapterHelper应该不陌生吧，它能帮助我们快捷地使用RecyclerView，这也是我为什么没有重新写一个库的原因，远离重复造轮子，追求创新，哪怕是微创新，都是很好的。话说我打了波广告，能不能给我点个star啊，开个玩笑。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;现在我来大致介绍下我新写的库，主要是项目中这种界面太多了，我们要有面向组件的思想，拿来就用，支持扩展。废话不多说，进入主题。

<!--more-->

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;先给大家欣赏几张优美的动态图，网差的朋友赶紧把demo下载下来，运行运行，看看效果。

![](/img/multi1.gif)

![](/img/multi2.gif)

![](/img/multi3.gif)

![](/img/multi4.gif)

欣赏了优美的动态图，我们来看看几个比较重要的集合

```
private SparseIntArray mLayouts;
private SparseIntArray mLevels;
private SparseArray<LevelData<T>> mLevelOldData;
protected List<T> mOldData;
```

1. mLayouts是根据用户注册的类型(下以type代替)用来存储资源文件id的。
2. mLevels是根据type用来存储模块的级别，比如A模块级别为0，B模块级别为1，那么A就在B的前面。<font color="red">（这里强调一下，大家在用的时候，数据的级别要从0开始，而头的级别，随便填一个负数就行了，当然可以用给出的默认头级别）</font>
3. mLevelOldData是记载了根据type存储下的LevelData，LevelData里面分别存储了，这个type下的数据集合以及头。
4. mOldData是用来存储老数据，也可以说还没刷新前的数据，也是用DiffUtil时的产物。

接下来介绍大致的用法

```
public class SampleAdapter extends MultiTypeRecyclerViewAdapter<MultiHeaderEntity, BaseViewHolder>
registerMoudle(TYPE_ONE, LEVEL_FIRST, R.layout.item_first);
registerMoudle(TYPE_TWO, LEVEL_FOURTH, R.layout.item_fourth);
registerMoudle(TYPE_THREE, LEVEL_SENCOND, R.layout.item_second);
registerMoudle(TYPE_FOUR, LEVEL_THIRD, R.layout.item_third);
registerMoudle(DEFAULT_TYPE_HEADER, DEFAULT_HEADER_LEVEL, R.layout.item_header);
registerMoudle(TYPE_HEADER_IMG, DEFAULT_HEADER_LEVEL, R.layout.item_header_img);
public void registerMoudle(int type, int level, @LayoutRes int layoutResId) {
    mLevels.put(type, level);
    mLayouts.put(type, layoutResId);
}
```

首先毫无疑问的是要继承我们的AsynMultiTypeAdapter(异步刷新)或者SynMultiTypeAdapter(同步刷新)，讲道理应该实现convert(K helper, T item)该方法，但是我项目中每个adapter基本都用到了position这个参数，可能会用到比如什么奇偶校验等，因此我重写了convert(K helper, T item, int position)，但没有设定为抽象级别，大家可以看自己想法实现哪个方法，然后在里面写渲染逻辑。registerMoudle(int type, int level, @LayoutRes int layoutResId)，该方法会将你页面的分的模块的类型以及该类型对应的级别和资源文件id存储下来，如果你想自己来处理异步刷新，请继承BaseMultiTypeRecyclerViewAdapter，然后用AsyncTask或者说RxJava等来实现，我直接用的线程池加Handler的方法。库中我会提供两个方法让你实现刷新的逻辑。

```
protected final DiffUtil.DiffResult handleRefresh(List<T> newData, T newHeader, int type, int refreshType) {
        int level = getLevel(type);
        int sum = 0;
        for (int i = 0; i < level; i++) {
        ......
}
protected void handleResult(DiffUtil.DiffResult diffResult) {
    diffResult.dispatchUpdatesTo(this);
    dismissDialog();
    isCanRefresh = true;
}
```
关于处理刷新的逻辑方法暂时不可重写，处理结果的可以。

下面是介绍如何把该adapter加载到我们的recyclerview中

```
recyclerView.setLayoutManager(new LinearLayoutManager(this) {
            @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        try {
            super.onLayoutChildren(recycler, state);
        } catch (IndexOutOfBoundsException e) {
            e.printStackTrace();
        }
    }
    @Override
    public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
        try {
            return super.scrollVerticallyBy(dy, recycler, state);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
});
```
大家最好是把这个方法try一下，官方BUG，小弟无能为力。

```
adapter.notifyMoudleDataAndHeaderChanged(list, new HeaderThirdItem("我是第三种类型的头", adapter.getRefreshHeaderId()), SampleAdapter.TYPE_FOUR);
public void notifyMoudleDataChanged(List<T> data, int type) {
    notifyMoudleChanged(data, null, type, REFRESH_DATA);
}
public void notifyMoudleHeaderChanged(T header, int type) {
    notifyMoudleChanged(null, header, type, REFRESH_HEADER);
}
public void notifyMoudleDataAndHeaderChanged(List<T> data, T header, int type) {
    notifyMoudleChanged(data, header, type, REFRESH_HEADER_DATA);
}
```

那剩下来得就是刷新了，我暂时只提供3个刷新方法，但是你也可以用BaseQuickAdapter(BaseRecyclerViewAdapterHelper库中的)中的刷新方法，关于这些方法怎么用，我相信机智的你，已经明白了，这里要说明一下，使用者不用关心头的什么level啊，id啊等，实体类对应什么布局，就差不多了，这里我提供adapter.getRefreshHeaderId()这个方法可以拿到头的id，默认从-1拿到long的最小值，当然你也可以自己给头设置id。

```
DiffUtil.calculateDiff(getDiffCallBack(mOldData, mData), isDetectMoves()).dispatchUpdatesTo(this);
//返回比较的callback对象，提供新老数据
protected DiffUtil.Callback getDiffCallBack(List<T> oldData, List<T> newData) {
    return new DiffCallBack<T>(oldData, newData);
}
//是否要移动
protected boolean isDetectMoves() {
    return true;
}
```

关于DiffUtil的用法，大家可以去看看，我这里不多介绍，但大家要注意的地方是，这个类在24.2.0版本才出现的。暂时提供DiffUtil.Callback和参数detectMoves，你可以实现自己的比较规则。这里有几个注意点，在比较的时候都是同步的，也就是说数据量比较大的时候，可能会出现anr，但是大多时候是没有问题的，我一般会做分页或者说刷新的时候都不会改动太大，我测试了一下，5000条数据，大概有2s左右的停滞。其实这都是我找的借口，当我写的时候我就注意到了，然后打算异步刷新，但是遇到了几个问题，大概涉及到这几个类的问题，GapWorker,ViewFlinger,ItemTouchHelper等，具体我也记不清了，大概crash原因都是recyclerview在回收的时候，数据的更新，还有view的回收时产生动画等，各种冲突产生crash，Google的大神们什么时候给我们一个完美的RecylcerView啊！但我会继续奋斗研究的，现在我提议是在比较数据的时候，可以加载进度框，简直是天才的想法。。。。。。好像哪里不对？主线程都阻塞了，你显示个蛋啊！话是这么说，我还是提供了对话框的功能，如果在用异步刷新的时候出现crash，最好把recyclerview的版本提至24.2.0版本。

接下来说说，我另外加的一个库StickyHeaderDecoration，我对它稍微做了点优化。库中的东西我没有全部集成，单单拿下了单粘性头的用法。大致用法如下：

```
public MultiTypeRecyclerViewAdapter(boolean isUseStickyHeader) {}
public void bindToRecyclerView(RecyclerView recyclerView) {}
```
在构造函数中传入是否使用粘性头，我提供了空参数构造，默认为false。但是你在setAdapter的时候用bindToRecyclerView()该方法，该方法的好处是你可以拿到recyclerView的引用，当然你可以自己给recyclerview进行addItemDecoration。

说一下这里的注意点，假设你用了GridLayoutManager，那么你想用粘性头，最好设置一个真头，且自己设置new StickyHeaderDecoration(adapter, true),这样粘性头就会遮挡住真头，看上去只有粘性头的效果，假如你要设置SpanSizeLookup，请在setAdapter之后设置，否则可能无效，至于为什么，我这边简单给出关键代码：

```
	
mAdapterHelper.reset();

```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到这里介绍的差不多了，稍微谈谈库的好处吧，最大的好处可能是快捷打造一个复杂布局，无论是粘性头，或者说是头部的布局，以及不用说的正常数据布局，都是可以自定化的。只要记住RecyclerView的核心是LayoutManager这个类，我个人这么认为。最近实在太忙，没时间写新奇的控件给大家，下次想给大家带来炫酷的3D控件，绝对真实感。希望大家能帮忙推广介绍下，点点star，就是我更新的最大动力，也是我不断进步的关键点。

库中依赖

```
compile fileTree(include: ['*.jar'], dir: 'libs')
compile 'com.android.support:recyclerview-v7:24.2.0'
compile 'com.github.CymChad:BaseRecyclerViewAdapterHelper:2.9.1'
compile 'com.android.support:appcompat-v7:24.2.0'
```

你可以自己设置相应的版本依赖，关于recyclerview的版本最好在24.2.0及以下，不然在异步刷新的时候可能会引起GapWork这货的异常问题，不用这家伙总没问题了把。

Gradle依赖

```
maven { url "https://jitpack.io" }
compile 'com.crazysunj:multitypeadapter:1.1.0'

```
这里要说一下的是上面的maven地址，这是因为我们的库依赖于BaseRecyclerViewAdapterHelper。

BaseRecyclerViewAdapterHelper库GitHub地址:
[https://github.com/CymChad/BaseRecyclerViewAdapterHelper](https://github.com/CymChad/BaseRecyclerViewAdapterHelper)

StickyHeaderDecoration库GitHub地址:
[https://github.com/edubarr/header-decor](https://github.com/edubarr/header-decor)

本库GitHub地址:
[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

本人博客地址:
[http://crazysunj.com/](http://crazysunj.com/)

