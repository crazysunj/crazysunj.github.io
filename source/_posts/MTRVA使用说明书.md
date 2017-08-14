---
title: MTRVA使用说明书
toc: true
date: 2017-08-14 11:23:21
tags: [Android,RecyclerView,Adapter]
---

## 介绍
MTRVA是对RecyclerViewAdapter的扩展，可以配合大多数的Adapter，核心功能是接管了Adapter中的资源和数据源。让用户真正的在Adapter中关心自己的业务逻辑。配合BRVAH更加简单，因此以下的示例都以BRVAH为例。

<!--  more-->
## 特点

* 简单快捷，可配合大多数Adapter
* 一行代码刷新单个type，支持一般的set，add，remove，clear等刷新
* 支持异步，高频率刷新，可扩展(如配合RxJava)
* 单个type支持Loading，Empty，Error页面切换
* 单个type支持header，footer
* 单个type支持展开和合拢(可设置合拢最小值)
* 支持注解生成类，减少工作量
* 支持刷新生命周期回调
* 兼容低版本RecyclerView

### 简单快捷，可配合大多数Adapter

Helper因为是跟Adapter配合，所以会增加无畏的工作量，那就是每个Adapter可能要创建一个Helper，所以要善于封装，复用等。这里提供BaseAdapter的封装示例。

```
public abstract class BaseAdapter<T extends MutiTypeTitleEntity, K extends BaseViewHolder, H extends RecyclerViewAdapterHelper<T, BaseAdapter>> extends BaseQuickAdapter<T, K> {

    protected H mHelper;

    public BaseAdapter(H helper) {
        super(helper.getData());
        mHelper = helper;
        mHelper.bindAdapter(this);
    }

    @Override
    protected K onCreateDefViewHolder(ViewGroup parent, int viewType) {
        return createBaseViewHolder(parent, mHelper.getLayoutId(viewType));
    }

    @Override
    protected int getDefItemViewType(int position) {
        return mHelper.getItemViewType(position);
    }

    public H getHelper() {
        return mHelper;
    }
}
```
在构造函数中，与Adapter绑定，向Adapter提供资源和相应position对应的itemType，基本工作就算完成。

Helper简单示例：

```
public class MyAdapterHelper extends AsynAdapterHelper<MutiTypeTitleEntity, BaseAdapter> {

    public MyAdapterHelper() {
        super(null);
    }

    @Override
    protected void registerMoudle() {

		...
        registerMoudle(ItemEntity3.TYPE_3)
                .level(2)
                .layoutResId(R.layout.item_3)
                .register();
		...
    }
}
```

AsynAdapterHelper是继承于RecyclerViewAdapterHelper，内部是采用Handler实现异步处理。下文会说其它的实现方法。registerMoudle是抽象方法，大家可以通过调用registerMoudle进行链式注册，基本的type，level，layoutId是必须的。level是type的序列等级，比如type1的level是0，type2的level是1，那么type1就在type2的前面。其它还有很多注册方法，例如loading,error等。

实战Adapter中运用：

```
public class MyAdapter extends BaseAdapter<MutiTypeTitleEntity, BaseViewHolder, MyAdapterHelper> {

    public MyAdapter() {
        super(new MyAdapterHelper());
    }

    @Override
    protected void convert(BaseViewHolder helper, MutiTypeTitleEntity item) {
        int itemType = item.getItemType();
        switch (itemType) {
            case ItemEntity1.TYPE_1:
                renderEntity1(helper, (ItemEntity1) item);
                break;
          	...
        }
    }

    private void renderEntity1(BaseViewHolder helper, ItemEntity1 item) {
        helper.setImageResource(R.id.item_1_img, item.getImg());
        helper.setText(R.id.item_1_title, item.getTitle());
        helper.setText(R.id.item_1_content, item.getContent());
        helper.setText(R.id.item_1_time, item.getTime());
        helper.setText(R.id.item_1_time_flag, item.getTimeFlag());
    }
	...
    public void notifyType1(List<ItemEntity1> itemEntity1s) {
        mHelper.notifyMoudleDataAndHeaderAndFooterChanged(entity1Header, itemEntity1s, entity1Footer, ItemEntity1.TYPE_1);
    }
	...
```

只要根据返回data的itemType进行判断，渲染相应的视图就行了。到此，与Adapter的配合就结束了，相当的简单。

### 一行代码刷新单个type，支持一般的set，add，remove，clear等刷新
回顾上面的示例代码，发现notifyType1方法，而里面调用的是Helper的notifyMoudleDataAndHeaderAndFooterChanged方法，这个方法，我们可以同时刷新data，header，footer。其它的还单刷data，或者header等，反正data,header,footer排列组合一下- -!同时还支持一般的set，add，remove，全局刷新等方法，具体可看方法注释，基本上每个方法都有相应注释。我们的刷新核心方法是利用diffutil实现了，但这个是24.2.0的时候出现的，下面会给出兼容方案。因为是底层是diffutil，所以要提供一个DiffCallBack，而它需要一个刷新比较的key，这里我们提供MultiTypeEntity接口，所有Bean实现它的id和itemType方法。

库中默认提供DiffCallBack：

```
@Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {

    T oldItem = mOldDatas.get(oldItemPosition);
    T newItem = mNewDatas.get(newItemPosition);
    return !(oldItem == null || newItem == null) && oldItem.getItemType() == newItem.getItemType();
}

@Override
public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {

    T oldItem = mOldDatas.get(oldItemPosition);
    T newItem = mNewDatas.get(newItemPosition);
    return oldItem.getId() == newItem.getId();
}
```

先用type过滤掉一部分，然后根据id比较。如果这没法满足你得需求，请重写getDiffCallBack方法，实现自己的DiffCallBack。

刷新支持2种模式，一种是常规的数据源，就是说List是乱的，没有按type连在一起，另一种就是标准的，两者可以相互切换，但是要注意，并不是随便切换，具体看注释。

### 支持异步，高频率刷新，可扩展(如配合RxJava)
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

之所以能实现高频率刷新而不错乱，是因为采用了串行的结构，内部有队列管理每次的刷新。

### 单个type支持Loading，Empty，Error页面切换
如果你想在单个type中进行Loading，Empty，Error之间的切换，请调用如下方法。

```
public void setLoadingAdapter(LoadingEntityAdapter<T> adapter) {
    mLoadingEntityAdapter = adapter;
}

public void setEmptyAdapter(EmptyEntityAdapter<T> adapter) {
    mEmptyEntityAdapter = adapter;
}

public void setErrorAdapter(ErrorEntityAdapter<T> adapter) {
    mErrorEntityAdapter = adapter;
}
```

这是创建资源的时候所需要的适配器，同RecyclerView的Adapter，而调用刷新方法：

```
public void notifyLoadingDataChanged(int type, @IntRange(from = 1) int dataCount);

public void notifyLoadingHeaderChanged(int type);

public void notifyLoadingDataAndHeaderChanged(int type, @IntRange(from = 1) int dataCount);

public void notifyMoudleEmptyChanged(T emptyData, int type);

public void notifyMoudleEmptyChanged(int type);

public void notifyMoudleErrorChanged(T errorData, int type);

public void notifyMoudleErrorChanged(int type);
```

如果Error和Empty调用带data的刷新方法，那么无需设置相应的Adapter，但是Loading必须要。Loading还有全局初始化方法。

```
 public void initGlobalLoadingConfig(LoadingConfig loadingConfig);

 public void notifyLoadingChanged(int type);

 public void notifyLoadingChanged();
```

如果是通过这种设置的，那么请调用下面的刷新方法，当然2种刷新方法可以结合使用，并不冲突。

### 单个type支持header，footer
在Helper注册资源的时候可以，添加。如：

```
registerMoudle(FirstOCEntity.OC_FIRST_TYPE)
            .level(0)
            .layoutResId(R.layout.item_first)
            .headerResId(R.layout.item_header)
            .footerResId(R.layout.item_footer)
            .isFolded(true)
            .minSize(3)
            .loading()
            .loadingHeaderResId(R.layout.layout_default_shimmer_header_view)
            .loadingLayoutResId(R.layout.layout_default_shimmer_view)
            .error()
            .errorLayoutResId(R.layout.layout_error_two)
            .empty()
            .emptyLayoutResId(R.layout.layout_empty)
            .register();
```

这是比较全的注册资源链。

### 单个type支持展开和合拢(可设置合拢最小值)
查看上面示例代码，发现isFolded和minSize方法，前者是设置是否合拢，默认为false，后者是合拢时候的最小值。使用方法：

```
public boolean isDataFolded(int type);

public void foldType(int type, boolean isFold);
```

isDataFolded方法是用来判断对应type所处的状态，foldType方法是用来展开和合拢用的。

### 支持注解生成类，减少工作量
这个东西不知道该不该介绍，主要用于简单注解，太多的并没有简单多数。在最新的版本1.7.0已结没有默认添加注解，如果需要使用则要另添加。具体添加方法。

```
com.crazysunj:multitypeadapter-annotation:xxx //版本与库的版本一致

apt 'com.crazysunj:multitypeadapter-compiler:xxx'
记得在项目gradle中添加

classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
主Module中添加

apply plugin: 'com.neenbedankt.android-apt'
```

具体使用方法戳[这里](http://crazysunj.com/2017/06/11/%E4%BC%98%E9%9B%85%E5%9C%B0%E5%88%B7%E6%96%B0RecyclerView%EF%BC%881-5-0%EF%BC%89/)

### 支持刷新生命周期回调

```
protected void onStart();
protected void onEnd();
public int getCurrentRefreshType();
```

需要使用生命周期回调方法的时候重写，再熟悉不过了，我知道回调的时候比较依赖当前type，因此提供了getCurrentRefreshType，用例比如需要在刷新前清除缓存，刷新后需要通知其他界面等。

### 兼容低版本RecyclerView
为什么产生兼容是因为我们底层用的是diffutil，然而这是在24.2.0出来了，低版本并没有。兼容方案如下：

```
compile 'com.crazysunj:diffutil:0.0.2' //diffutil支持库，24.2.0以上并不需要，不然会冲突
```
这还没完，在版本22的RecyclerViewAdapter并没有带3个参数的notifyItemRangeChanged方法，因此，想要兼容22(RecyclerView在22.0.0出现)，得重写Helper的getListUpdateCallback方法，并在onChanged方法回调中使用带两个参数的方法。如下：

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

## 注意点
### Type 取值范围

* level [0,+∞)
* data类型 [0,1000)
* header类型 [-1000,0)
* loading-data类型 [-2000,-1000)
* loading-header类型 [-3000,-2000)
* error类型 [-4000,-3000)
* empty类型 [-5000,-4000)
* footer类型 [-6000,-5000)

### 差值常量

```
//header类型差值
public static final int HEADER_TYPE_DIFFER = 1000;
//loading-data类型差值
public static final int LOADING_DATA_TYPE_DIFFER = 2000;
//loading-header类型差值
public static final int LOADING_HEADER_TYPE_DIFFER = 3000;
//error类型差值
public static final int ERROR_TYPE_DIFFER = 4000;
//empty类型差值
public static final int EMPTY_TYPE_DIFFER = 5000;
//footer类型差值
public static final int FOOTER_TYPE_DIFFER = 6000;
```

### 其他

```
protected int getPreDataCount();
```
获取数据前条目数量，保持数据与Adapter一致，比如Adapter的position=0添加的是headerView，如果你的Adapter有这样的条件，请重写该方法或者自己处理数据的时候注意，否则数据混乱，甚至崩溃都是有可能的。

调用刷新方法的时候请注意刷新模式，有些方法只支持相应刷新模式，需要注意的都加有check语句。

关于entity的id为long类型是考虑刷新效率，你大可采用多种属性的加密（MD5）的hashCode或者就是普通hashCode作为主键。倘若还支持不了你的数据(出现哈希冲突，并不是没刷新，可打日志调试)，就自定义DiffCallback(可参数demo)。

具体可参考Demo，建议把helper封装在Adapter中。

## gradle依赖

```
compile 'com.crazysunj:multitypeadapter:1.7.0'
compile 'com.android.support:recyclerview-v7:xxx'
```

### 传送门
Github：[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

博客：[http://crazysunj.com/](http://crazysunj.com/)

谷歌邮箱：twsunj@gmail.com

QQ邮箱：387953660@qq.com

