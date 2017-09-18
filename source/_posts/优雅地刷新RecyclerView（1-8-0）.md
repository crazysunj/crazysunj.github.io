---
title: 优雅地刷新RecyclerView（1.8.0）
toc: true
date: 2017-09-17 15:47:04
tags:
---
## 前言
这次更新说多不多，说少不少。

## 更新内容
* 去除部分过时类和方法
* addData方法支持标准模式
* 优化资源的注册
* 添加2个功能方法

<!--  more-->
### 去除部分过时类和方法
这次把1.7.0版本标记为过时的方法和类给去除了，这些类主要用于以前粘性头的，但这属于扩展，并不是我们的核心内容，因此把它移除，很早几个版本已经有提到。例如我们熟悉的MultiHeaderEntity。

### addData方法支持标准模式
以前是不支持的，因为这个确实比较麻烦，就算是现在支持，但在添加集合的情况下，支持并不是很完善，比如你添加集合，它可能正好是当前type的末尾，又是下个level的一部分，甚至下下个level的一部分，例如当前集合为AABBE，而add的position正好是B之后且添加的集合为BCCD，而C，D的level小于E，有这种复杂的情况存在，当前支持添加的集合为当前position的type。

### 优化资源的注册
以前资源注册有内部类直接引用外部类，尽量避免这样写法，现在用静态内部类代替。

### 添加2个功能方法
一个可以说是优化，一个是真正的添加。

```
public void clearMoudle(int... type);
public void remainMoudle(int... type);
```

前者是清除type数组中所有type，例如原集合为ABBCDDE，type数组为BDE，那么调用该方法后就为AC。后者是保留type数组中的所有type，同上例，那么结果为BBDDE。

## 结束语
<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView，Adapter的流派很多，我们更关注于RecyclerView的优雅刷新（数据源处理）。

如果对本库还不是很了解的同学可以到我的github查看具体版本进行了解。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**

## 传送门

Github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com

QQ邮箱:387953660@qq.com


