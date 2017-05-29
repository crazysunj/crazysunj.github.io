---
title: 优雅地刷新RecyclerView（1.4.0）
toc: true
date: 2017-05-29 15:39:55
tags:
---

## 前言
RecyclerView的viewType增多，逻辑变复杂，几个月后，你确定还能理清思路吗？假设我们服务端是多个接口返回数据，你确定能正确刷新相应type吗？想一个RecyclerView高效快捷管理整个界面吗？你还在使用notifyDataSetChanged无脑刷新吗？你想单个viewType在loadingView，dataView，errorView，emptyView自如切换吗？你还在为刷新导致数据错乱而烦恼吗？如果你迟疑了，那你不妨试试本库。

<!--  more-->
## 特点

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

## 效果
### 线性排布
![](/img/adapterHelper1.gif)

### 方格排布
![](/img/adapterHelper3.gif)

### 关键字高亮
![](/img/adapterHelper4.gif)

### 错误页面
![](/img/adapterHelper5.gif)

### 空页面
![](/img/adapterHelper7.gif)

### 高频率刷新
![](/img/adapterHelper6.gif)


## 更新内容

* 支持add,set,remove,全局等方法刷新
* 支持加载相应type空页面
* 优化清除数据
* 优化异常命名
* 支持标准(一个type对应一个集合)和混合(一般的多类型集合)自如切换(自动排序集合)
* ......

### 支持add,set,remove,全局等方法刷新

```
public void notifyDataSetChanged(List<? extends T> newData, @RefreshMode int refreshMode);

public void notifyDataSetChanged(List<? extends T> newData);

public void notifyDataSetChanged();

public void notifyDataByDiff(List<? extends T> newData, @RefreshMode int refreshMode);

public void notifyDataByDiff(List<? extends T> newData);

public void addData(int position, T data);

public void addData(T data);
 
public T removeData(int position);

public void setData(int position, @NonNull T data);

public void addData(int position, List<? extends T> data);

public void addData(List<? extends T> newData);
```

两参数notifyDataSetChanged方法可在切换刷新模式时候刷新全局(数据源为传参)，单参数默认刷新模式为当前刷新模式，无参数刷新当前刷新模式且当前数据源。

notifyDataByDiff用法同notifyDataSetChanged，区别在于刷新的时候采用DiffUtil。

其他方法就不做介绍了吧？

### 支持加载相应type空页面

```
public void notifyMoudleEmptyChanged(EmptyEntity emptyData, int type);

public void notifyMoudleEmptyChanged(int type);
```
用法同error。

### 优化清除数据
在清除数据前都做了空判断。

### 优化异常命名
库中抛出的异常都为自定义，很友爱，嘿嘿。

### 支持标准(一个type对应一个集合)和混合(一般的多类型集合)自如切换(自动排序集合)

```
public void switchMode(@RefreshMode int mode, boolean isSort);

public void switchMode(@RefreshMode int mode);

private List<T> initStandardNewData(List<T> newData);
```

isSort标志是否要排序，如果你能确保数据的有序性，可填false，默认为true。排序会调用initStandardNewData进行排序，上面的notifyDataSetChanged和notifyDataByDiff也可切换刷新模式。

### 其他
还有一些细节优化就不作介绍了。

## 注意点
### Type 取值范围

* level [0,+∞)
* 数据类型 [0,1000)
* 头类型 [-1000,0)
* shimmer数据类型 [-2000,-1000)
* shimmer头类型 [-3000,-2000)
* error类型 [-4000,-3000)
* empty类型 [-5000,-4000)

### 差值常量

```
//头类型差值
public static final int HEADER_TYPE_DIFFER = 1000;
//shimmer数据类型差值
public static final int SHIMMER_DATA_TYPE_DIFFER = 2000;
//shimmer头类型差值
public static final int SHIMMER_HEADER_TYPE_DIFFER = 3000;
//错误类型差值
public static final int ERROR_TYPE_DIFFER = 4000;
//空类型差值
public static final int EMPTY_TYPE_DIFFER = 5000;
```

### 其他

```
protected int getPreDataCount();
```
获取数据前条目数量，保持数据与adapter一致，比如adapter的position=0添加的是headerview，如果你的adapter有这样的条件，请重写该方法或者自己处理数据的时候注意，否则数据混乱，甚至崩溃都是有可能的。

调用刷新方法的时候请注意刷新模式，有些方法只支持相应刷新模式，需要注意的都加有check语句。

关于entity的id为long类型是考虑刷新效率，你大可采用多类型的UUID的hashcode或者就是普通hashcode作为主键（注意缓存）。倘若还支持不了你的数据，就自定义DiffCallback。

具体可参考Demo。

## 结束

<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView,Adapter的流派很多，我们更关注于数据的优雅刷新。

如果对本库还不是很了解的同学可以到我的github查看更多版本进行了解。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**
## gradle依赖

```
compile 'com.crazysunj:multitypeadapter:1.4.0'
```

## 传送门

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com

QQ邮箱:387953660@qq.com










