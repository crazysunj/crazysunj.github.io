---
title: 减少30%工作量的AdapterHelper
toc: true
date: 2017-05-07 13:56:46
tags: [Android,RecyclerView,Adapter,helper,multiType,RxAndroid,RxJava,HighLight,DiffUtil]
---

## 由来
随着业务需求越来越多，越来越复杂，相应的UI界面随之变化。我相信RecyclerView大家已经用的不陌生了，但是它的繁琐构建，确实是件头疼的事情，特别是viewType特别多，逻辑特别复杂的情况，过几个月，你确定还能理清思路吗？假设我们服务端是多个接口返回数据，你确定能正确刷新相应type吗？想一个RecyclerView高效快捷管理整个界面吗？你还在使用notifyDataSetChanged无脑刷新吗？如果你迟疑了，那你不妨试试本库。

<!--  more-->

## 特点

* 与Adapter为组合关系
* 一行代码刷新相应viewType
* 支持facebook的shimmer加载效果
* 支持粘性头
* 支持异步刷新，可扩展(如配合RxAndroid)

## 效果
### Rx线性排布
![](/img/adapterHelper1.gif)

### 一般线性排布
![](/img/adapterHelper2.gif)

### 方格排布
![](/img/adapterHelper3.gif)

### 关键字高亮
![](/img/adapterHelper4.gif)

## 解析
### 与Adapter为组合关系
最初是直接继承Adapter，但是Java限制单继承，现在Adapter分支太多，可能每个公司都有自己的封装Adapter，避免不必要的修改代码，这里提供组合关系管理Adapter。

```
public CommonHelperAdapter(RecyclerViewAdapterHelper<T> helper) {

    mData = helper.getData();
    helper.bindAdapter(this);
    mHelper = helper;
}
```

用法很简单，Adapter的数据集合mData依赖于helper的mData，helper绑定相应Adapter，Adapter剩下的实现方法用helper去实现就算完成工作了(具体的可参考DEMO)。是不是很简单？

关于本地实现自己的helper那就更简单了，在实现的Helper中注册相应的数据，具体如下。

```
public void registerMoudle(@IntRange(from = 0, to = 999) int type, @IntRange(from = 0) int level,
                               @LayoutRes int layoutResId, @LayoutRes int headerResId) 

public void registerMoudleWithShimmer(@IntRange(from = 0, to = 999) int type, @IntRange(from = 0) int level,
                                          @LayoutRes int layoutResId, @LayoutRes int headerResId,
                                          @LayoutRes int shimmerLayoutResId, @LayoutRes int shimmerHeaderLayoutResId)
```

上边是类型注册（头和数据同时注册），当然肯定还有单数据注册，带有Shimmer的注册可以提供Shimmer效果的loading，具体布局大家可以根据设计师要求修改，这里提供了默认的头和数据loading布局。

到这里构造工作算是完成了，毫无难度，只能说毫无难度。这里有个缺陷就是type和level的范围限制，具体如下：

* level [0,+∞)
* 数据类型 [0,1000)
* 头类型 [-1000,0)
* shimmer数据类型 [-2000,-1000)
* shimmer头类型 [-3000,-2000)

### 一行代码刷新相应viewType

```
public void notifyMoudleDataAndHeaderChanged(List<T> data, T header, int type)

public void notifyShimmerDataAndHeaderChanged(int type, @IntRange(from = 1) int dataCount) 
```

还有比这更简单？再自己封装一下，参数都不用这么多- -！其他还提供单头和单数据的刷新，简单的我都不知道说啥了，下一个。

### 支持facebook的shimmer加载效果

上面的注册和刷新都提到了Shimmer这个关键字，假设你注册了带Shimmer的，那么你可以使用带Shimmer的刷新，具体的loading布局可以在注册的时候提供，关于Shimmer效果，在实现自己的ViewHolder的时候，去实现Shimmer的效果，然后在自己需要的时机调用，非常灵活。如下是DEMO中的例子：

```
public void startAnim() {

    if (itemView instanceof ShimmerFrameLayout) {
        ((ShimmerFrameLayout) itemView).startShimmerAnimation();
    }
}
```

ShimmerFrameLayout的具体配置可在xml或者ViewHolder的构造函数去配置，具体用法->[传送门](https://github.com/facebook/shimmer-android)

<font color="red">友情提示：</font>这里提供简单的ViewHolder封装（CommonViewHolder）和Adapter封装（CommonHelperAdapter），大家不要再写一堆的ViewHolder了。

### 支持粘性头

```
recyclerView.addItemDecoration(new StickyHeaderDecoration(adapter));

public StickyHeaderDecoration(StickyHeaderAdapter adapter)

public StickyHeaderDecoration(StickyHeaderAdapter adapter, boolean renderInline)
```

是的，你没有看错，一行代码搞定，但前提是你的Adapter要实现StickyHeaderAdapter。

```
public interface StickyHeaderAdapter<T extends RecyclerView.ViewHolder> {

    long getHeaderId(int position);

    T onCreateHeaderViewHolder(ViewGroup parent);

    void onBindHeaderViewHolder(T holder, int position);
}
```

### 支持异步刷新，可扩展(如配合RxAndroid)

能快速找到要刷新的数据，这里借用了DiffUtil，具体用法我就不介绍了，但是有个缺陷就是如果数据量过大的时候，计算的时候很费时，因此把它放在线程中不影响用户操作。库中的异步刷新实现是传统的handler方法，但是我把计算和处理结果的接口提供了，大家可以打造自己的异步处理，这里举个DEMO中栗子，利用RxAndroid(这里是2.0)实现：

```
Flowable.just(new HandleBase<MultiHeaderEntity>(newData, newHeader, type, refreshType))
                .onBackpressureDrop()
                .observeOn(Schedulers.computation())
                .map(new Function<HandleBase<MultiHeaderEntity>, DiffUtil.DiffResult>() {
                    @Override
                    public DiffUtil.DiffResult apply(@NonNull HandleBase<MultiHeaderEntity> handleBase) throws Exception {
                        return handleRefresh(handleBase.getNewData(), handleBase.getNewHeader(), handleBase.getType(), handleBase.getRefreshType());
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<DiffUtil.DiffResult>() {
                    @Override
                    public void accept(@NonNull DiffUtil.DiffResult diffResult) throws Exception {
                        handleResult(diffResult);
                    }
                });
```

**温馨提示:** 如果你的Adapter起始并不是你的数据集合，比如你设置headerLayout等，请重写getListUpdateCallback()，在相应的位置刷新，以免数据错乱异常。关于实体类的id为long类型，考虑的是比较效率，大家可以用hashcode，注意冲突，如果勉强不妨自定义DiffCallBack。

### 关键字高亮

```
public static CharSequence handleKeyWordHighLight
    (String originStr, String keyWord, @ColorInt int hightLightColor)
```
这里我提供一个静态方法给大家，传原始的字符串，关键字和高亮颜色即可，很多人问这个，索性在库中提供了。

## 结束

说实话，我都不知道说啥，实现方便快捷，可以配合大多数RecyclerViewAdapter，我最想说的可能就是我希望大家能使用我的库，找出相应的缺陷，不管是代码逻辑还是性能也好，大家共同进步。大家动一动手指头给个star，谢谢了。

## gradle依赖

```
compile 'com.crazysunj:multitypeadapter:1.2.0'
```

## 感谢

[shimmer-android](https://github.com/facebook/shimmer-android)

## 传送门

github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

邮箱:twsunj@gmail.com


