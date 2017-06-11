---
title: 优雅地刷新RecyclerView（1.5.0）
toc: true
date: 2017-06-11 13:37:36
tags: [Android,RecyclerView,Adapter,Helper]
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
* 支持注解生成类，减少工作量
* 支持刷新生命周期回调

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

### loading页面
![](/img/adapterHelper8.gif)

## 更新内容

* 支持配置全局loading页面
* 支持刷新生命周期回调
* 支持注解生成类，减少工作量
* ......

### 支持配置全局loading页面

```
public void initGlobalLoadingConfig(LoadingConfig loadingConfig);

public void notifyShimmerChanged(int type);

public void notifyShimmerChanged();

helper.initGlobalLoadingConfig(new LoadingConfig.Builder()
                .setLoading(true, SimpleHelper.TYPE_ONE, 3, true)
                .setLoading(true, SimpleHelper.TYPE_TWO, 2)
                .setLoading(true, SimpleHelper.TYPE_THREE, true)
                .setLoading(true, SimpleHelper.TYPE_FOUR, 4, true)
                .build());
```

调用initGlobalLoadingConfig方法配置每个type的loading页面属性，上面为demo中的调用语句，其中LoadingConfig采用构建者模式，那么怎么用呢？带参数notifyShimmerChanged方法只会刷新相应type的loading页面（数据源为配置时的），如果想不用配置数据源可调用原来的
notifyShimmerChanged方法。不带参数的notifyShimmerChanged方法是根据配置刷新全局。


### 支持刷新生命周期回调

```
protected void onStart();

protected void onEnd();

public int getCurrentRefreshType();
```

需要使用生命周期回调方法的时候重写，再熟悉不过了，我知道回调的时候比较依赖当前type，因此提供了getCurrentRefreshType，用例比如需要在刷新前清除缓存，刷新后需要通知其他界面等。

### 支持注解生成类，减少工作量

```
@AdapterHelper()
String superObj() default "...";
String entity() default "...";
String adapter() default "...";

@BindType(）
int level();

@BindAllType()
int level();
int layoutResId();
int headerResId() default 0;
int loadingLayoutResId() default 0;
int loadingHeaderResId() default 0;
int errorLayoutResId() default 0;
int emptyLayoutResId() default 0;

@BindDefaultType()
int headerResId() default 0;
int loadingLayoutResId() default 0;
int loadingHeaderResId() default 0;
int errorLayoutResId() default 0;
int emptyLayoutResId() default 0;

@PreDataCount()

@BindLayoutRes()
int[] type();

@BindHeaderRes
int[] type();

@BindLoadingLayoutRes
int[] type();

@BindLoadingHeaderRes
int[] type();

@BindErrorLayoutRes
int[] type();

@BindEmptyLayoutRes
int[] type();
```

仔细看完的同学内心独白：WTF？放下菜刀，听我慢慢讲来，注解PreDataCount我就不用说了吧，注意点一直强调；该注解以下的注解更强调多个type共用了同个资源；注解BindDefaultType用来注解只有一种type的时候；注解AdapterHelper用来注解需要生成helper的adapter，其中参数superObj为生成helper的父类，entity为实体类，adapter为绑定adapter，参数类型为类引用路径。如何生成，不用我说了吧，我猜你已经轻车熟路，生成类名暂时为Adapter类型+Helper。

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

具体可参考Demo，建议把helper封装在Adapter中。

## 结束

<font color="red">你的意见，你的建议，你的star，你的分享，一直是我前进的动力。</font>现在关于LayoutManager，RecyclerView,Adapter的流派很多，我们更关注于数据的优雅刷新。

如果对本库还不是很了解的同学可以到我的github查看具体版本进行了解。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**
## gradle依赖

```
compile 'com.crazysunj:multitypeadapter:1.5.0'
apt 'com.crazysunj:multitypeadapter-compiler:1.5.0'
```
如果想用注解生成可依赖，记得在项目gradle中添加

```
classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
```

主Module中添加

```
apply plugin: 'com.neenbedankt.android-apt'
```

## 传送门

Github:[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客:[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱:twsunj@gmail.com

QQ邮箱:387953660@qq.com


