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

## 引入库

本库很好地解决了上述问题，具体使用这里就不介绍了，下面给出库的部分功能：

* 简单快捷，可配合大多数Adapter
* 一行代码刷新相应viewType
* 支持粘性头
* 支持异步刷新，可扩展(如配合RxAndroid)
* 支持高频率刷新(流畅,异步执行)
* 支持加载facebook的shimmer效果loading页面
* 支持加载相应type错误页面
* 支持加载相应type空页面
* 支持标准(一个type对应一个集合)和混合(一般的多类型集合)自如切换(自动排序集合)
* 支持集合set,add,remove,clear等操作刷新
* 支持注解生成类，减少工作量
* 支持刷新生命周期回调

 <img src="/img/weiju.jpg" width = "180" height = "160"/>

## 结束

<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView,Adapter的流派很多，我们更关注于数据的优雅刷新。

如果对本库还不是很了解的同学可以到我的github查看具体版本进行了解，每个版本都有详细的介绍。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**

## 传送门

Github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com

QQ邮箱:387953660@qq.com
