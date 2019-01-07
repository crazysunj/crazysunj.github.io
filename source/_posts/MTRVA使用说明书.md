---
title: MTRVA使用说明书
toc: true
date: 2017-08-14 11:23:21
tags: [Android,RecyclerView,Adapter]
---
![](/img/mtrva_logo.png)
## 介绍
MTRVA是对RecyclerViewAdapter的扩展，支持大多数的Adapter，核心功能是接管了Adapter中的资源和数据源，处理了大量与业务无关的数据计算，让用户真正的在Adapter中关心自己的业务逻辑。

支持高灵活、低耦合、健壮性、高效性以及MVP。
## 架构
![](/img/mtrva_architecture.png)
<!--  more-->
## gradle依赖

```
// support最后一个版本
implementation 'com.crazysunj:multitypeadapter:2.3.1'
implementation 'com.android.support:recyclerview-v7:xxx'

AndroidX的同学：
implementation 'com.crazysunj:multitypeadapter:2.4.0'
implementation 'androidx.recyclerview:recyclerview::xxx'
```

后续版本更新只支持AndroidX，毕竟Google将停止维护support包，没有更新的小伙伴得抓紧了，不过项目大的同学慢慢来。
## 特点

* 使用简单快捷，支持大多数Adapter(高灵活、低耦合)
* 一行代码刷新(附动画)单个level(可对应多个type)
* 支持增删改查操作(健壮性)
* 支持异步，高频率，链式刷新，可扩展(如配合RxJava，高效性)
* 单个level支持Loading(加载)，Empty(空)，Error(错误)页面切换
* 单个level支持header，footer
* 单个level支持展开和合拢(可设置合拢最小值)
* 支持加载全局Loading(加载)页面
* 支持注解生成类，减少工作量
* 支持刷新生命周期回调
* 兼容低版本RecyclerView
* 进阶用法，比如在多种页面之间自由切换

### 使用简单快捷，支持大多数Adapter(高灵活、低耦合)

Helper是Adapter的扩展类，会增加无畏的工作量，每个Adapter可能要创建一个Helper，所以要善于封装，复用等。这里提供BaseHelperAdapter的封装示例以示Helper与和Adapter的配合。

```
public abstract class BaseHelperAdapter<T extends MultiTypeEntity, VH extends BaseViewHolder, H extends RecyclerViewAdapterHelper<T>> extends RecyclerView.Adapter<VH> {

    protected Context mContext;
    protected LayoutInflater mLayoutInflater;
    protected List<T> mData;
    protected H mHelper;

    public BaseHelperAdapter(H helper) {
        mData = helper.getData();
        helper.bindAdapter(this);
        mHelper = helper;
    }

    @Override
    public int getItemViewType(int position) {
        return mHelper.getItemViewType(position);
    }
    ...
    @NonNull
    @Override
    public VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        return createBaseViewHolder(parent, mHelper.getLayoutId(viewType));
    }
    ...
}
```
在构造函数中，与Adapter绑定，向Adapter提供资源和相应position对应的itemType(注意，一定要保证内外position一致，如果你是用的第三方Adapter，可能position等于0的时候是你Adapter的headView)，基本工作完成。这里省略无关代码，想看全部可到demo中查看。

Helper简单示例：

```
public class TestLevelAdapterHelper extends AsynAdapterHelper<MultiTypeTitleEntity> {
    ...
    public TestLevelAdapterHelper(List<MultiTypeTitleEntity> data) {
        super(data);
    }

    @Override
    protected void registerModule() {
        registerModule(LEVEL_FIRST)
                .type(TypeOneItem.TYPE_ONE)
                .layoutResId(R.layout.item_first)
                .type(TypeTwoItem.TYPE_TWO)
                .layoutResId(R.layout.item_second)
                .headerResId(R.layout.item_header)
                .footerResId(R.layout.item_footer)
                .loading()
                .loadingHeaderResId(R.layout.layout_default_shimmer_header_view)
                .loadingLayoutResId(R.layout.layout_default_shimmer_view)
                .register();
        ...
    }
    ...
}
```

AsynAdapterHelper是继承于RecyclerViewAdapterHelper，内部是采用Handler实现异步处理。下文会说其它的实现方法。registerModule是抽象方法，大家可以通过调用registerModule进行链式注册，基本的level是必须的。level是type的序列等级，比如type1的level是0，type2的level是1，那么type1就在type2的前面。其它还有很多注册方法，例如loading,error等。

实战Adapter中运用：

```
public class TestLevelAdapter extends BaseHelperAdapter<MultiTypeTitleEntity, BaseViewHolder, TestLevelAdapterHelper> {
    ...
    public TestLevelAdapter(List<MultiTypeTitleEntity> data) {
        super(new TestLevelAdapterHelper(data));
        ...
    }

    @Override
    protected void convert(BaseViewHolder holder, MultiTypeTitleEntity item) {
        final int itemType = item.getItemType();
        switch (itemType) {
            case TypeOneItem.TYPE_ONE:
            case TypeTwoItem.TYPE_TWO:
            case TYPE_LEVEL_FIRST_HEADER:
            case TYPE_LEVEL_FIRST_FOOTER:
                mFirstItemConvert.convert(holder, item);
                break;
            ...
        }
    }
	...
    public void notifyLevelFirst(MultiTypeTitleEntity header, List<MultiTypeTitleEntity> data, MultiTypeTitleEntity footer) {
        mHelper.notifyModuleDataAndHeaderAndFooterChanged(data, header, footer, TestLevelAdapterHelper.LEVEL_FIRST);
    }
	...
}
```

只要根据返回data的itemType或者说item的类型进行判断，渲染相应的视图就行了。到此，与Adapter的配合就结束了，相当的简单。

### 一行代码刷新(附动画)单个level(可对应多个type)
回顾上面的示例代码，发现notifyLevelFirst方法，而里面调用的是Helper的notifyModuleDataAndHeaderAndFooterChanged方法，这个方法，我们可以同时刷新data，header，footer。其它的还单刷data，或者header等，反正data,header,footer排列组合一下- -!同时还支持一般的set，add，remove，全局刷新等方法，具体可看方法注释，基本上每个方法都有相应注释。我们的刷新核心方法是利用DiffUtil实现了，但这个是24.2.0的时候出现的，下面会给出兼容方案。因为是底层是DiffUtil，所以要提供一个DiffCallBack，而它需要一个刷新比较的key，这里我们提供MultiTypeEntity接口，所有Bean实现它的id和itemType方法。由于底层是DiffUtil，所以刷新的时候是局部刷新并带有动画，原理可以看我这篇文章[《BRVAH+MTRVA，复杂？不存在的》](http://crazysunj.com/2017/08/14/BRVAH-MTRVA%EF%BC%8C%E5%A4%8D%E6%9D%82%EF%BC%9F%E4%B8%8D%E5%AD%98%E5%9C%A8%E7%9A%84/)。

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

上面一节了解到资源的注册，其实可以多个type一起注册在一个level中，比如像这样：

```
registerModule(LEVEL_FIRST)
        .type(TypeOneItem.TYPE_ONE)
        .layoutResId(R.layout.item_first)
        .type(TypeTwoItem.TYPE_TWO)
        .layoutResId(R.layout.item_second)
        ...
        .register();
```

刷新支持2种模式，一种是常规的数据源，就是说List是乱的，没有按level连在一起，另一种就是标准的，两者可以相互切换，但是要注意，并不是随便切换，具体看注释。

此外，很多用户习惯直接操作数据，然后再调用我们的notify方法，但这样其实是不会通知Adapter的，因为引用没变，对比的是同一引用。故这里提供了3种刷新方法(底层均调用Adapter的notifyItemRangeChanged方法)。

```
public void notifyDataChanged(); //全局刷新一遍
public void notifyDataChanged(@NonNull T data);//刷新单个数据
public void notifyDataChanged(int level);//刷新整个level
```

### 支持增删改查操作(健壮性)
这个比较好理解，除了上面的add，remove，set（增删改）以外，你还可以进行查的操作，比如这样：

```
public List<T> getData();

public LevelData<T> getDataWithLevel(int level);

public LevelData<T> getDataWithType(int type);
```
getData可以帮助我们查到原始数据源，而getDataWithType/getDataWithLevel可以帮助我们根据type/level查到对应的数据，而现在是可以对查到的数据进行直接操作修改，但大家尽量不要在这里直接操作数据。

这里提供了强力的删操作：

```
public void clearModule(int... level);

public void remainModule(int... level);
```
从方法命名上我们可以知道clearModule可以清楚多个level的数据，而remainModule是保留多个level的数据，意味着没有保留的都会被删除。

库中的增删改查做了很多兼容性，增强代码的健壮性，本来你以为会报错，结果没报错，可以具体查看代码中的逻辑。

### 支持异步，高频率，链式刷新，可扩展(如配合RxJava，高效性)
能快速找到要刷新的数据，这里借用了DiffUtil，具体用法我就不介绍了，但是有个缺陷就是如果数据量过大的时候，计算的时候很费时，因此把它放在线程中不影响用户操作。库中的异步刷新实现是传统的handler方法，但是我把计算和处理结果的接口提供了，大家可以打造自己的异步处理，这里举个DEMO中例子，利用RxAndroid(这里是2.0)实现：

```
Flowable.just(new HandleBase<MultiHeaderEntity>(newData, newHeader, type, refreshType))
        .onBackpressureDrop()
        .observeOn(Schedulers.computation())
        .map(new Function<HandleBase<MultiHeaderEntity>, DiffUtil.DiffResult>() {
            @Override
            public DiffUtil.DiffResult apply(@NonNull HandleBase<MultiHeaderEntity> handleBase) throws Exception {
                return handleRefresh(handleBase.getNewData(), handleBase.getNewHeader(), handleBase.getLevel(), handleBase.getRefreshType());
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

如果你是喜欢链式的同学，那么你有福了，MTRVA也支持链式调用。例如这样：

```
AdapterHelper.with(level)
        .data(data|list)
        .header(header)
        .footer(footer)
        .into(helper);

AdapterHelper.with(level)
        .empty(|empty)
        .into(helper);

AdapterHelper.with(level)
        .error(|error)
        .into(helper);

AdapterHelper.with(level)
        .loading()
        .header()
        .data(count)
        .into(helper);

AdapterHelper.action()
        .clear()
        .remainModule(level)
        .clearModule(level)
        .set(position,data)
        .remove(position,count)
        .remove(position)
        .add(position,list)
        .add(position,data)
        .add(list)
        .add(data)
        .all(list)
        .all(data)
        .into(helper);
```
最后一种虽然调起来很爽，但是得注意，执行顺序是自下而上。

### 单个level支持Loading(加载)，Empty(空)，Error(错误)页面切换
如果你想在单个level中进行Loading，Empty，Error之间的切换，请调用如下方法。

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
public void notifyLoadingDataChanged(int level, @IntRange(from = 1) int dataCount);

public void notifyLoadingHeaderChanged(int level);

public void notifyLoadingDataAndHeaderChanged(int level, @IntRange(from = 1) int dataCount);

public void notifyModuleEmptyChanged(T emptyData, int level);

public void notifyModuleEmptyChanged(int level);

public void notifyModuleErrorChanged(T errorData, int level);

public void notifyModuleErrorChanged(int level);
```

如果Error和Empty调用带data的刷新方法，那么无需设置相应的Adapter，但是Loading必须要。

### 单个level支持header，footer
在Helper注册资源的时候可以，添加。如：

```
registerModule(LEVEL_FIRST)
        .type(FirstOCEntity.OC_FIRST_TYPE)
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

### 单个level支持展开和合拢(可设置合拢最小值)
查看上面示例代码，发现isFolded和minSize方法，前者是设置是否合拢，默认为false，后者是合拢时候的最小值。使用方法：

```
public boolean isDataFolded(int level);

public void foldType(int level, boolean isFold);
```

isDataFolded方法是用来判断对应level所处的状态，foldType方法是用来展开和合拢用的。

### 支持加载全局Loading(加载)页面
如果你得项目页面有刚进入需要展示加载页面，可以参考[首页Demo](https://www.pgyer.com/sOVg)。

```
 public void initGlobalLoadingConfig(LoadingConfig loadingConfig);

 public void notifyLoadingChanged(int level);

 public void notifyLoadingChanged();
```

如果是通过这种设置的，那么请调用下面的刷新方法，而在loading的小节中也有相应的刷新方法，细心的朋友会发现只是少了数量这个参数，因为在全局初始化的时候已经设置好了，因此这2种刷新方法可以结合使用，并不冲突。

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

具体使用方法戳[这里](http://crazysunj.com/2017/06/11/%E4%BC%98%E9%9B%85%E5%9C%B0%E5%88%B7%E6%96%B0RecyclerView%EF%BC%881-5-0%EF%BC%89/)，2.0.0版本布局架构改变，因此相应的注解也有所改变，但使用方法不变，可与之前的注解对比使用。

**注意点:**如果你使用的是as3.0，那么请移除上面的的插件依赖，且apt改为annotationProcessor。

### 支持刷新生命周期回调

```
protected void onStart();
protected void onEnd();
public int getCurrentRefreshLevel();
```

需要使用生命周期回调方法的时候重写，再熟悉不过了，我知道回调的时候比较依赖当前level，因此提供了getCurrentRefreshLevel，用例比如需要在刷新前清除缓存，刷新后需要通知其他界面等。

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

### 进阶用法
由于我们是多个level的定义，因此有没有想过，level最高级为headerView，level最低级为footerView，而且完全自定义封装；这还不是重点，作为一个Activity拥有一个RecyclerView就够了，很多情况，我们可能会有数据为空的情况，那么就需要一个展示空数据的页面，同理，错误页面也是这样。其实这，本库也能做到。比如像这样：

```
registerModule(LEVEL_SWITCH)
        .type(SwtichType.TYPE_A)
        .layoutResId(R.layout.item_switch_type)
        .type(SwtichType.TYPE_B)
        .layoutResId(R.layout.item_switch_type)
        .type(SwtichType.TYPE_C)
        .layoutResId(R.layout.item_switch_type)
        .type(SwtichType.TYPE_D)
        .layoutResId(R.layout.item_switch_type)
        .register();
        
        
public void notifyA() {
    mHelper.notifyDataByDiff(new SwtichType(SwtichType.TYPE_A, "我是typeA"));
}

public void notifyB() {
    mHelper.notifyDataByDiff(new SwtichType(SwtichType.TYPE_B, "我是typeB"));
}

public void notifyC() {
    mHelper.notifyDataByDiff(new SwtichType(SwtichType.TYPE_C, "我是typeC"));
}

public void notifyD() {
    mHelper.notifyDataByDiff(new SwtichType(SwtichType.TYPE_D, "我是typeD"));
}
```

代码共展示了4种类型，一个空和错误页面根本不在话下，甚至你可以定义更多种异常情况的页面。使用notifyDataByDiff方法可以在多个列表页面自由切换。

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
保持数据position与Adapter一致，比如Adapter的position=0添加的是headerView，如果你的Adapter有这样的条件，请重写该方法或者自己处理数据的时候注意，否则数据混乱。

调用刷新方法的时候请注意刷新模式，当前支持两种模式。

关于entity的id为long类型是考虑刷新效率，倘若支持不了你的数据，例如服务器返回的主键是字符串类型的，你又不想把String转化为long，就自定义DiffCallback(可参数demo)。

建议把helper封装在Adapter中，不刷新时考虑一下DiffCallback的比较key，最常见的可能是同一引用引起的，建议接口返回的时候可以用刷新方法，因为都新对象，如果是针对单个或者部分数据操作，可以采用普通的add，set，remove等方法。当然了，如果是2.4.0版本以上的直接使用AdapterHelper类去刷新，会兼容此场景。

不必在意库名MultiType这个字眼，当时自己也陷入了type的思维定势陷阱中并取了这样的名字（后缀为Adapter是因为早期是继承Adapter，但这样并不灵活），虽然讲道理也并没有错，RecyclerView不就是拿来支持多type的吗？但我们负责的是Adapter中的资源和数据源，利用规范的数据去驱动UI，像什么多类型Adapter，动画Adapter，下拉刷新上拉加载Adapter等等都是支持的。

**欢迎大家的star(fork)和反馈(可发issues或者我的邮箱）。**

## 传送门
Github：[https://github.com/crazysunj/MultiTypeRecyclerViewAdapter](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)

谷歌邮箱：twsunj@gmail.com

QQ邮箱：387953660@qq.com

