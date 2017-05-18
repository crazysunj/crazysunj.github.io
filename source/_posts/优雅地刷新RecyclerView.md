---
title: 优雅地刷新RecyclerView
toc: true
date: 2017-05-18 14:15:42
tags: [Android,RecyclerView,Adapter,Helper,Error]
---

## 前言

还是那句话，RecyclerView的viewType增多，逻辑变复杂，几个月后，你确定还能理清思路吗？假设我们服务端是多个接口返回数据，你确定能正确刷新相应type吗？想一个RecyclerView高效快捷管理整个界面吗？你还在使用notifyDataSetChanged无脑刷新吗？你想单个viewType在loadingView,dataView,errorView自如切换吗？如果你迟疑了，那你不妨试试本库。

## 特点

* 与Adapter为组合关系，可配合大多数Adapter
* 一行代码刷新相应viewType
* 支持facebook的shimmer加载效果
* 支持粘性头
* 支持异步刷新，可扩展(如配合RxAndroid)
* 支持加载相应type错误页面
* 支持高频率刷新(流畅,异步执行)


## 效果
### 线性排布
![](/img/adapterHelper1.gif)

### 方格排布
![](/img/adapterHelper3.gif)

### 关键字高亮
![](/img/adapterHelper4.gif)

### 刷新错误页面
![](/img/adapterHelper5.gif)

### 高频率刷新
![](/img/adapterHelper6.gif)

## 更新内容
* 支持刷新type错误页面(可自定义)
* 支持同时刷新多个type(异步，高频率)
* 链式注册资源
* 支持刷新单个数据
* 提供helper的清除单个type，清除整个界面api

### 支持刷新type错误页面(可自定义)

```
public void notifyMoudleErrorChanged(ErrorEntity errorData, int type);

public void notifyMoudleErrorChanged(int type);
```

一行代码搞定，前者提供实体类是考虑有些用户需要根据实体数据属性去更新，因此错误页面的layoutId是用户提供的。

### 支持同时刷新多个type(异步，高频率)

```
 //刷新队列，支持高频率刷新
private Queue<HandleBase<T>> mRefreshQueue;
```

这里采用的是队列的形式管理刷新，提供清空队列的Api。

### 链式注册资源


```
registerMoudle(@IntRange(from = 0, to = 999) int type)
                .level(@IntRange(from = 0) int level)
                .layoutResId(@LayoutRes int layoutResId)
                .headerResId(@LayoutRes int headerResId)
                .loading()
                .loadingLayoutResId(@LayoutRes int loadingLayoutResId)
                .loadingHeaderResId(@LayoutRes int loadingHeaderResId)
                .error()
                .errorLayoutResId(@LayoutRes int errorLayoutResId)
                .register();
```

由于参数越来越多，这里采用了较为流行的链式注册，内部通过ResourcesManager管理所有资源。

**注：**原来的注册方式已设置为过时，请及时更新，不出2个版本将移除。


### 支持刷新单个数据

```
public void notifyMoudleDataAndHeaderChanged(T data, T header, int type)
```
可能某个type只有一个实体数据管理着整个type

```
public void notifyMoudleDataAndHeaderChanged(List<? extends T> data, T header, int type)
```

可传T的子类集合

### 提供helper的清除单个type，清除整个界面api

```
/**
 * 清除单个type数据
 *
 * @param type 数据类型
 */
public void clearMoudle(int type);

/**
 * 清除所有数据
 */
public void clear();
```

### 使用注意点
 type 取值范围
 
 * level [0,+∞)
 * 数据类型 [0,1000)
 * 头类型 [-1000,0)
 * shimmer数据类型 [-2000,-1000)
 * shimmer头类型 [-3000,-2000)
 * error类型 [-4000,-3000)

常量差值

```
//头类型差值
public static final int HEADER_TYPE_DIFFER = 1000;
//shimmer数据类型差值
public static final int SHIMMER_DATA_TYPE_DIFFER = 2000;
//shimmer头类型差值
public static final int SHIMMER_HEADER_TYPE_DIFFER = 3000;
//错误类型差值
public static final int ERROR_TYPE_DIFFER = 4000;
```

## 结束

库多多少少也更新几个版本了，不管你是刚入门也好，还是转行的也好，还是老鸟，甚至是大牛，恳请你们能反馈该库的缺陷，我会更近，让更多的人体验优雅的刷新机制，可以把意见发给我的谷歌邮箱或者说QQ邮箱(就是我的QQ），下面给出，请动一动手指分享出去。还有一点要说的就是现在关于LayoutManager，RecyclerView,Adapter的流派很多，我们更关注于数据的优雅刷新。

## gradle依赖

```
compile 'com.crazysunj:multitypeadapter:1.3.0'
```

## 传送门

github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com
QQ邮箱:387953660@qq.com