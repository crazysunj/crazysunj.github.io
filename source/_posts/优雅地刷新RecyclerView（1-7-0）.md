---
title: 优雅地刷新RecyclerView（1.7.0）
toc: true
date: 2017-08-14 11:27:59
tags: [Android,RecyclerView,Adapter]
---

## 前言

距离前一个版本已经挺久了，这次的更新主要是规范我们的库，让他成为真正的意味着专门为RecyclerView服务的一个刷新（资源和数据源处理）的库，废话不多说直接看这次我们的更新内容。

<!--  more-->
## 更新内容

* 去除部分类和资源
* 部分类和方法过时
* 兼容版本低版本的RecyclerView
* 修改部分类和方法结构
* 修复初始化全局Loading数据bug
* 增加新demo

### 去除部分类和资源

放心，兼容方案都给大家想好了
#### 去除粘性包
这次去除了粘性包，大家可以依赖我的另一个库[RecycylerViewItemDecoration](https://github.com/crazysunj/RecycylerViewItemDecoration)，包名和用法并没有变

#### 去除Shimmer包
Shimmer包的github地址[shimmer-android](https://github.com/facebook/shimmer-android)

#### 去除apt注解和RecyclerView依赖
原来都是库中直接依赖的，现在改为想用就依赖，更加灵活，这里给出apt注解的依赖方法。

```
com.crazysunj:multitypeadapter-annotation:xxx //版本与库的版本一致
```

#### 去除资源
res目录下的资源都已经移去，兼容方法就是下载demo，资源全部移到demo的res目录下了，跟几个小伙伴讨论了下，真的没必要放太多的东西。

### 兼容版本低版本的RecyclerView
由于diffutil这个支持库在24.2.0加入的，因此低于这个版本的都得gg，这里提供了兼容库[RecyclerViewDiffUtil](https://github.com/crazysunj/RecyclerViewDiffUtil)（注意，如果是24.2.0以上的并不需要依赖），版本22的RecyclerViewAdapter并没有带3个参数的notifyItemRangeChanged方法，因此，想要兼容22(RecyclerView在22.0.0出现)，得重写Helper的getListUpdateCallback方法，并在onChanged方法回调中使用带两个参数的方法。

例如像这样：

```
@Override
    protected ListUpdateCallback getListUpdateCallback(final BaseAdapter adapter) {
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
            adapter.notifyItemRangeChanged(position + preDataCount, count);
        }
    };
}
```

### 修改部分类和方法结构
其实方法就一个，原来的刷新方法notifyMoudleHeaderAndDataAndFooterChanged更新为notifyMoudleDataAndHeaderAndFooterChanged，主要是为保持Data->Header->Footer的顺序。

很巧，类也只有一个，CommonHelperAdapter类的泛型结构改变，具体：

```
CommonHelperAdapter<T extends MultiTypeEntity, K extends CommonViewHolder, H extends RecyclerViewAdapterHelper<T, CommonHelperAdapter>> extends RecyclerView.Adapter<K>

CommonHelperAdapter(H helper)
```

### 修复初始化全局Loading数据bug
原来如果先初始化LoadingData全局，然后刷新全局LoadingData，再刷新部分type，会发现loading页面还在，因为在刷新全局Loading数据时，并没有把loadData缓存下来，核心方法就当它不存在的情况，只会在原来的基础上添加，并不会移除或者替换LoadingData。

### 增加新demo
请参考这篇文章[《BRVAH+MTRVA，复杂？不存在的》](/2017/08/14/BRVAH-MTRVA，复杂？不存在的/)

## 结束语
这次的更新说多不多，说少也不少，只是希望能让大家减少更多的工作量，不再为刷新而烦恼，能够有更多的时间陪媳妇。

<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView，Adapter的流派很多，我们更关注于RecyclerView的优雅刷新（数据源处理）。

如果对本库还不是很了解的同学可以到我的github查看具体版本进行了解。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**

## 传送门

Github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com

QQ邮箱:387953660@qq.com
