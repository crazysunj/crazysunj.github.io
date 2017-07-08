---
title: 优雅地刷新RecyclerView（1.6.0）
toc: true
date: 2017-07-08 13:00:51
tags: [Android,RecyclerView,Adapter,Helper,Expand,Footer]
---

## 前言

最近确实有点忙，导致状态有点差，关于这次的更新的想法我早就有了，基本顺着以前打下的草稿写了下来，也希望能够更好地帮助大家刷新RecyclerView，让它变得更简单，不再去理会那繁琐的刷新逻辑，有数据源就能刷新，就这么简单。这样想想，好像充满活力！

<!--  more-->
## 更新内容

* 单个type支持footer
* 去掉Error，Empty，Loading实体类，增加Adapter构建
* 支持单Type展开和合拢
* 修改刷新核心方法算法
* 修改了Loading刷新方法名，Shimmer改为Loading
* 去掉过时注册方法，现在只能通过链式注册
* 其它

## 使用与部分解析
### 单个type支持footer

```
public TypesManager footerResId(@LayoutRes int footerResId) {
    this.footerResId = footerResId;
    return this;
}
```
注册footer布局资源，同header。

```
public void notifyMoudleFooterChanged(T footer, int type)；

public void notifyMoudleHeaderAndFooterChanged(T header, T footer, int type)；

public void notifyMoudleDataAndFooterChanged(List<? extends T> data, T footer, int type)；

public void notifyMoudleDataAndFooterChanged(T data, T footer, int type)；

public void notifyMoudleHeaderAndDataAndFooterChanged(T header, List<? extends T> data, T footer, int type)；

public void notifyMoudleHeaderAndDataAndFooterChanged(T header, T data, T footer, int type)；
```

新增的关于footer的刷新方法，使用过的朋友我相信已经很熟悉了，第一次的也没关系，demo中有详细的使用方法。

其它修改地方基本是兼容footer，如单个type存储数据实体类的数据结构，修改如下：

```
public LevelData(List<T> data, T header, T footer) {
    this.data = data;
    this.header = header;
    this.footer = footer;
}
```
真的很庆幸当时把header和data分开了，现在增加footer就简单多了，可能当时有想到，过会儿又忘了。哈哈。

footer就到这里了，主要跟header差不多，就不多介绍了。

### 去掉Error，Empty，Loading实体类，增加Adapter构建

关于这一点，我也不知道做的对还是不对，原来Loading实体类是包私有的，外界并不用关心，只要告诉你要几个，我们就会帮你创建。而Error和Empty的刷新实体类都必须继承于我们提供的相应实体类。从而一看，并不灵活，更致命的一点，如果你想用loading，error，empty，那么helper的泛型似乎失去了意义，你只能用MultiHeaderEntity。如果你想用三者的默认构建功能（只要告诉我们你的type，我们将会自动为你构造相应数据，关于loading暂时只能设置Adapter），那么必须得用设置三者的Adapter。设置方法如下：

```
public void setLoadingAdapter(LoadingEntityAdapter<T> adapter);

public interface LoadingEntityAdapter<T> {

    T createLoadingEntity(int type);

    T createLoadingHeaderEntity(int type);

    /**
     * loadingEntity是复用的
     *
     * @param loadingEntity 复用entity
     * @param position      所在索引，头的索引为-1
     */
    void bindLoadingEntity(T loadingEntity, int position);
}

public void setEmptyAdapter(EmptyEntityAdapter<T> adapter);

public interface EmptyEntityAdapter<T> {

    T createEmptyEntity(int type);
}

public void setErrorAdapter(ErrorEntityAdapter<T> adapter);

public interface ErrorEntityAdapter<T> {

    T createErrorEntity(int type);
}
```

### 支持单Type展开和合拢

这并不是简单的展开和合拢，一般的，合拢只留下一个头。但是我们这里可以做到合拢到几行，什么意思呢？简单来说假如我们的合拢值为3，单个type数据有6行，当我们操作为合拢的时候，只会剩下3行数据，展开又是6行数据。这并不是什么神奇的地方，刷新无非是对数据的操作，对集合的理解。这里有个小tips，在设置数据源的时候不需要自己去处理，你只要把原来的，完整的数据源传过来就行了，接下来的就交给我们吧。

比如你直接传一个List给我们，

```
mLevelOldData.put(level, new LevelData<T>(data, header, footer));
......
ResourcesManager.AttrsEntity attrsEntity = mResourcesManager.getAttrsEntity(level);
if (attrsEntity == null || !attrsEntity.isFolded || data.size() <= attrsEntity.minSize) {
    addData.addAll(data);
} else {
    addData.addAll(data.subList(0, attrsEntity.minSize));
}
```
从上面代码，我们可以看出，原数据，我们会存储在mLevelOldData.put中，而给Adapter的数据将是已经截取过的。

那么如何展开和合拢呢？

```
public void foldType(int type, boolean isFold);
```
第一个参数就是你要操作的type，后一个参数，如果为false，那么就是展开，反之为true，就是合拢。但仅限在数据页面调用，下同（比如空页面，错误页面调用，均无意义）。

如果我们想获取某个type的展开合拢状态，可不可以呢？放心，no problem！

```
public boolean isDataFolded(int type);
```
每次只要单个type填充新的数据，那么标志位将会重置为注册时的值。

### 修改刷新核心方法算法
增加了footer和展开合拢功能，总是避免不了会修改该方法。

主要增加如下：

```
//获取修改索引的时候，计算footer
if (data.getFooter() != null) {
    sum++;
}

//操作老数据的footer
T oldItemFooter = oldLevelData.getFooter();
if (oldItemFooter != null) {
    if (isRefreshFooter) {
        mNewData.remove(oldItemFooter);
    } else {
        footer = oldItemFooter;
    }
}

//添加展开合拢数据
if (!(newData == null || newData.isEmpty()) && isRefreshData) {
    data = newData;
    ResourcesManager.AttrsEntity attrsEntity = mResourcesManager.getAttrsEntity(level);
    if (attrsEntity == null || !attrsEntity.isFolded || data.size() <= attrsEntity.minSize) {
        mNewData.addAll(header == null ? positionStart : positionStart + 1, data);
    } else {
        mNewData.addAll(header == null ? positionStart : positionStart + 1, data.subList(0, attrsEntity.minSize));
    }
}
```

### 修改了Loading刷新方法名，Shimmer改为Loading
确切地来说，shimmer只是loading视图的一个效果，不能因为它是facebook的就取代了功能名（手动滑稽）。

### 其他
* 增加Loading全部刷新的状态，可在刷新生命周期中获取

```
//刷新loading全部
public static final int REFRESH_TYPE_LOAD_ALL = -3;
```

* 刷新生命周期增加CallSuper注解，里面的内容很重要，避免不必要的错误

```
/**
 * 刷新生命周期-开始
 */
@CallSuper
protected void onStart() {
    mIsCanRefresh = false;
}

/**
 * 刷新生命周期-结束
 */
@CallSuper
protected void onEnd() {
    mIsCanRefresh = true;
    mCurrentType = REFRESH_TYPE_IDLE;
}
```

* 异步Helper增加重置刷新队列方法

```
/**
 * 重置刷新队列
 */
public void reset() {
    clearQueue();
    cancelFuture();
}
```

* 去掉了构建Loading集合的sticky支持参数

因为Loading数据的属性完全是由你自己设置，如果要用sticky，跟data，header等设置一样就行了。

```
void bindLoadingEntity(T loadingEntity, int position);
```

* 修改diffcallback的判断比较逻辑

把areItemsTheSame中的id比较移到了areContentsTheSame中，当时也确实没注意到这点，虽然我对id的获取做了缓存，但还是没必要多余的调方法，首先我们用type比较可以过滤掉绝大部分，把范围限定在同一个type中，然后进行id的比较，看到日志的打印数减少，心情豁然开朗。这里有个小tips，不是这两个方法或者说三个方法结果一样就行了，判断先后是会影响最后的刷新方法的调用的。我知道你不信，去试试就知道什么是浪费时间。

## 结束语
这次的更新说多不多，说少也不少，只是希望能让大家减少更多的工作量，不再为刷新而烦恼，能够有更多的时间陪媳妇。

<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView，Adapter的流派很多，我们更关注于数据的优雅刷新。

如果对本库还不是很了解的同学可以到我的github查看具体版本进行了解。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**

## 传送门

Github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com

QQ邮箱:387953660@qq.com


