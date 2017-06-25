---
title: 打造RecyclerView全能刷新Helper
toc: true
date: 2017-06-11 13:41:36
tags: [Android,RecyclerView,Adapter,Helper]
---

## 前言
我们已经越来越离不开RecyclerView，业务的复杂导致界面的复杂，而RecyclerView无疑是最佳之选。RecyclerView如何打造出复杂的布局，如何打造高性能，多功能的RecyclerView，如何打造高复用，灵活的Adapter等，这些问题，这篇文章都不涉及（手动滑稽）。这篇文章讲的是似乎被忽略的RecyclerView的刷新，各种姿势的刷新，妈妈再也不用担心产品的各种奇葩刷新了。

<!--  more-->
## 库的由来
项目开发周期短让我们没时间自己去实现或者封装库，最好有现成的库（切记重复造轮子），但我找了一圈也没找到，这就尴尬了。只能自己动手丰衣足食，好的东西不能自己享用，因此开源，大家一起享受。

本库沿着几条需求进行展开，现在一一分析。

 <img src="/img/jihe.jpg" width = "180" height = "140"/>

### RecyclerView的viewType增多，逻辑变复杂，几个月后，你确定还能理清思路吗？

多type的实现，常规做法就是每个type对应一个集合，然后在adapter中处理这些集合，一个两个还好，更多呢？而且在开发周期比较短的情况，总是避免不了后期的重构，最尴尬的是重构的时候，自己写的是啥忘了（多注释啊）。在这里，你根本不需要考虑刷新前数据处理逻辑。

### 假设我们服务端是多个接口返回数据，你确定能正确刷新相应type吗？

我们总是避免不了服务端会分接口返回，多个接口合并处理吧，影响效率，接口响应时间不一致，如果需求是只刷新某个type呢？因此我们的全局刷新梦想也破灭了。我们不得不完美地把控每个position对应的ViewType,一旦出错，可能导致条目错乱，甚至崩溃，这太影响用户体验了。在这里，刷错type，你打我。

### 想一个RecyclerView高效快捷管理整个界面吗？

其实这条跟我们的刷新库并无多大关系，更考验的是对RecyclerView的理解，但是管理整个界面总是避免不了有各种各样的type。

### 你还在使用notifyDataSetChanged无脑刷新吗？

什么？无脑？但是简单好用啊，而且很少出错。notifyDataSetChanged方法刷新是没有动画的，给人感觉很生硬，使用带动画的刷新又容易出错，还有一个更致命的缺点是不需要刷新的也强制刷新了。在这里，吸纳了各自的优点。

### 你想单个viewType在loadingView，dataView，errorView，emptyView自如切换吗？

厉害了，我的哥，服务端多接口返回数据的时候，可能某个接口出错了，我们总不能整个页面出错吧？不显示又太low了，搞个轻提示吧，用户还不一定看见。在这里，你想怎么切就怎么切。

### 你还在为刷新导致数据错乱而烦恼吗？

这种错误往往出现在使用Adapter的范围型刷新，比如notifyItemRangeInserted，用户一刷新，发现有2个一样的type，死的心都有了，这其实还算好，如果你的app拥有切换条件的功能，比如区域等，不同区域不同数据，可是，用户切换的时候，2个区域的数据都出现了。我的天，被开除的节奏。在这里，你可以保住你饭碗。

![](/img/rizi.jpeg)

## 现在

此情此景，我想吟诗一首，啊，我想要一个能解决上述问题且能配合我们项目的Adapter的库。少年，往这里看，不知道这些服务，您还满意吗？

* 简单快捷，可配合大多数Adapter
* 一行代码刷新相应viewType
* 支持粘性头
* 支持异步刷新，可扩展(如配合RxAndroid)
* 支持高频率刷新(流畅,异步执行)
* 支持加载facebook的shimmer效果loading页面
* 支持加载相应type错误页面
* 支持加载相应type空页面
* 支持标准(一个type对应一个集合)和混合(一般的多类型集合)自如切换(自动排序集合)
* 支持集合set，add，remove，clear等操作刷新
* 支持注解生成类，减少工作量
* 支持刷新生命周期回调

 <img src="/img/weiju.jpg" width = "180" height = "160"/>

## 未来

看完上面的同学可能要问了，到底怎么使用啊？实现原理又是什么啊？放心，我会一一给你解答。

### 使用方法

使用方法在这里不便给出，实在太多了，但我肯定不是那种不负责任的人，对于每次大更新，更新点以及使用方法我都会写一篇文章，你可以到我的博客看，也可以到我的github上看。对于每个类，每个方法基本都有注释，这点你可以放心。上面说的完全够你对库的使用手到擒来。

### 实现原理
本库的刷新基本都是围绕着DiffUtil展开，那么我们从这里切入。

#### DiffUtil
DiffUtil内部采用的Eugene W. Myers’s difference 算法，通过传入新老数据集，计算二者间的差异，再调用相应的方法进行刷新：

```
adapter.notifyItemRangeInserted(position, count);
adapter.notifyItemRangeRemoved(position, count);
adapter.notifyItemMoved(fromPosition, toPosition);
adapter.notifyItemRangeChanged(position, count, payload);
```
这样一来，我们可以跟notifyDataSetChanged说88啦。那么新老数据集怎么来？

#### 数据集
以下type新老数据集，简称数据集，整个数据集，简称数据源。

老数据源好说，不就是adapter里面的数据源嘛。新数据源呢？如果是整一个数据集，直接拿来用就是了，可我现在只想更新数据源里面的某个type的数据集，最好有个以type为key的map集合管理着每个type的数据集，当我们要更新数据集的时候，可以直接掏出老数据集。I have a old type list，I have a new type list，new type list！

到这里，我想了两个办法。其一，更新map集合中需要更新的type的value为新数据集，然后再遍历组合成新数据源。其二，copy一份老数据源，先移除老数据集，再添加新数据集。这里先不分析孰优孰劣，我选择了后者。

无论是第一种还是第二种都面临一个问题，数据集的位置。假设现在我们有3种type，不妨为A，B，C，产品要求顺序为A，B，C（如果B数据集为空，那么就为A，C，以此类推），但你现在可能出现C，B，A，老铁，这不是打篮球，恐怕你又得被开除啊。这种优先级的概念，我们见得太多了，比如进程的优先级，有序广播的优先级等等。所以这里我们引入了level的概念，且把map的key改为level，举个比较明显问题的栗子，遍历map的时候，你还是不知道谁前谁后。那你会说，我根据level找到type，再根据type找到数据集不就好了？很不幸，我们这里，level跟type是一对多的关系，比如上面说的A，它可能用来显示正常的数据，万一产品说如果数据出错，我们需要有错误页面（错误页面级别是type），那岂不是GG？

map的key改为level后，两种方法的实现思路很明显了。这里贴出第二种的代码。

```
mNewData.addAll(mData);//copy老数据源

mNewData.removeAll(oldItemData);
mNewData.remove(oldItemHeader);//移除老数据集

mNewData.addAll(oldHeader == null ? positionStart : positionStart + 1, newData);

mNewData.add(positionStart, newHeader);//添加新数据集      
```

细心的同学可能发现，positionStart怎么来的？它是靠map遍历得到的，但它是不完全遍历（这里涉及到最优时间复杂度）。

到这里，新老数据源都有了，剩下的就是交给diffutil去更新UI。

```
DiffUtil.DiffResult result = DiffUtil.calculateDiff(getDiffCallBack(mData, mNewData), isDetectMoves());

diffResult.dispatchUpdatesTo(getListUpdateCallback(mAdapter));
 
mData.addAll(mNewData);//切记，切记，一定要保持内外数据源一致     
```
这里我们看到getDiffCallBack(mData, mNewData)，isDetectMoves()，getListUpdateCallback(mAdapter)。先来说说isDetectMoves()方法，返回值作为DiffUtil静态方法calculateDiff的第二个参数，这个参数官方是这么说的。

```
If move detection is enabled, it takes an additional O(N^2) time where N is the total number of added and removed items. If your lists are already sorted by the same constraint (e.g. a created timestamp for a list of posts), you can disable move detection to improve performance.
```

大概意思是说，如果该参数为true，那么在计算的时候，会额外增加O(N^2) 的时间复杂度，N为移动的数量（增加和删除），如果列表已经按约束设计了（不需要调整），建议填false。可能这么说比较抽象，官方也给出了测试数据（调试数据为一组随机UUID字符串，运行在Android版本型号为M的Nexus 5X），我们来看看吧。

```
100项中10项修改：平均值：0.39毫秒，中位数：0.35毫秒
100项中100项修改：平均值：3.82毫秒，中位数：3.75毫秒
100个项目中100个修改（不移动）：平均值：2.09毫秒，中位数：2.06毫秒
1000项中50项修改：平均值：4.67毫秒，中位数：4.59毫秒。
1000个项目中50个修改（不移动）：平均值：3.59毫秒，中位数：3.50毫秒
1000项中200项修改：平均值：27.07毫秒，中位数：26.92毫秒
1000个项目中200个修改（不移动）：平均值：13.54毫秒，中位数：13.36毫秒
```

接下来我们来看看getListUpdateCallback方法，这个比较好理解，它作为dispatchUpdatesTo方法的参数。返回值ListUpdateCallback是对计算数据的回调。我们来看看库的默认实现。

```
protected ListUpdateCallback getListUpdateCallback(final RecyclerView.Adapter adapter) {

    final int preDataCount = getPreDataCount();

    return new ListUpdateCallback() {
        @Override
        public void onInserted(int position, int count) {
            adapter.notifyItemRangeInserted(position + preDataCount, count);
        }

        @Override
        public void onRemoved(int position, int count) {
            adapter.notifyItemRangeRemoved(position + preDataCount, count);
        }

        @Override
        public void onMoved(int fromPosition, int toPosition) {
            adapter.notifyItemMoved(fromPosition + preDataCount, toPosition + preDataCount);
        }

        @Override
        public void onChanged(int position, int count, Object payload) {
            adapter.notifyItemRangeChanged(position + preDataCount, count, payload);
        }
    };
}
```

比官方的默认实现多了preDataCount变量，该变量是为了保证内部数据position与adapter的position一致。同学们可以通过重写getPreDataCount方法改变值。

最后一个getDiffCallBack方法，这个较为复杂，但是熟悉了也还好，我这里简单介绍一下，感兴趣的可以到官方文档看，官方是最权威的，小弟的英文也不太好，所以。。。那个。。。言归正传，该方法的返回值为抽象类DiffUtil.Callback，共有5个方法。

```
public abstract int getOldListSize();

public abstract int getNewListSize();

public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);

public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);

@Nullable
public Object getChangePayload(int oldItemPosition, int newItemPosition) {
    return null;
}
```
前面两个我就不说了，见名知意，中间2个，其实也很明显，第三个看名字是说判断2个条目是否相同的，恭喜你答对了，这个地方我们一般判断两个条目的“主键”，如果返回true才会调areContentsTheSame方法，看名字就是让我们判断条目中的内容是否一样，可以判断其中一项，也可以判断多项，甚至全部。最后一个方法getChangePayload，是配合Adapter中

```
public void onBindViewHolder(VH holder, int position, List<Object> payloads)
```
Android源码中该方法是调两个参数的方法，那么第三个参数怎么来的呢？我们回上去看看getListUpdateCallback方法，里面有这么一个方法

```
@Override
public void onChanged(int position, int count, Object payload) {
    adapter.notifyItemRangeChanged(position + preDataCount, count, payload);
}
```

卧槽，我懂了，是的，就是这么一回事。那它有什么用呢？比如像这样，

```
@Nullable
@Override
public Object getChangePayload(int oldItemPosition, int newItemPosition) {
	Bundle payload = new Bundle();
	payload.putInt("MONEY", 998);
	return payload;
}
```

最后在adapter回调方法onBindViewHolder中取出Bundle，根据Bundle来局部更新，不用全部走一遍。

好了，这里栗子用的Bundle，大家可以看到数据类型其实是Object，之所以用Bundle是因为我们要跟随谷歌爸爸的脚步，Bundle在Android数据通信这一块作用还是很大的。

#### 异步

上面result与diffResult不一致是我采用了两个方法，原因是DiffUtil造成的。

```
If the lists are large, this operation may take significant time so you are advised to run this on a background thread, get the DiffUtil.DiffResult then apply it on the RecyclerView on the main thread.
```

大概意思说，数据量太大的时候，计算最好放到子线程，计算结束再到主线程更新UI。没毛病，因此拆成2个方法，如果是异步，则先调用前者，再切到主线程调用后者。但这其实是线程不安全的，与Android引入Handler更新UI有点类似（不理解的同学，可以去看看Handler的相关文章）。这里我们选择了串行的方法并引入了以单链表结构的队列来管理每次刷新的数据源。

```
HandleBase<T> pollData = mRefreshQueue.poll();
if (pollData != null) {
    startRefresh(pollData);
} else {
    onEnd();
}
```

我们这里没有Looper的概念，因为我知道它什么时候开始，什么时候结束。

以上就是本库的核心原理啦，其它还有像什么资源管理（个人认为设计的不够好），数据的创建，模式的切换，生命周期的回调等。感兴趣的同学可以看看源码。

### 展望未来

有一句话是这么说的，当局者迷旁观者清。如果得到你们的支持，我相信他会越来越完善，能够支持更多用户的项目，他并不是我的，是大家的，有米娜的陪伴，他一定能够更好。刚把带！

<img src="/img/jiayou.jpg" width = "180" height = "160"/>

## 结束语

<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView，Adapter的流派很多，我们更关注于数据的优雅刷新。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**

## 传送门

Github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)
