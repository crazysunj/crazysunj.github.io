---
title: RecyclerView组件化
toc: true
date: 2019-11-02 17:56:49
tags: [MTRVA, Component]
---

## 前言
最近又收到了[MTRVA](https://github.com/crazysunj/MultiTypeRecyclerViewAdapter)的问题反馈，中间有一段时间很忙没更新，反馈很少，前段时间更新了`2.5.0`版本，好像又活了，看来我们的MTRVA还是很有影响...力的，啪~谁打我，不好意思，屌丝的YY心理又犯了。言归正传，反馈比较多的一个问题是这样的，虽然**MTRVA**很好用，大大减少了数据处理的工作量，但还是避免不了在**onBindViewHolder**中写一堆itemType判断。其实到这里，说明你还没明白**MTRVA**的作用，它只做资源的接管和数据的处理，至于你的**XML**怎么写，**LayoutManager**怎么写，**onBindViewHolder**怎么写等等都不在库的范围内。刚刚列举的功能能做到的库有很多，但我们的**MTRVA**都能与之匹配，这就是强大的`扩展性`。

![](/img/image_niao_huang.jpeg)

emmm，我前思后想，这里把不外传的秘诀传给你们，真是便宜你们了。这个其实很简单，**onBindViewHolder**里面有一堆的逻辑就是因为你没有抽离出来，没有组件化的思想。

***注：本文介绍组件化是建立在MTRVA的基础上。***

## 组件化
### 定义
这里简单说一下我对组件化的理解，为什么说简单，因为我tm也不知道我说的对不对。接触到这个概念肯定是来自别人，表达的意思大概是，Android中每个module库算一个组件，很具体的，可能这是对的，毕竟真理都是有条件的，只是我对其有一丝丝的怀疑。

我所理解的组件化就是`分治`。一个整体可以分成多个独立的部分，这些独立的部分取其一部分又可以组成新的整理，而这些独立的部分称之为组件。在我们Android中整体可以大到一个项目，组件可以小到一个控件。而我们今天的主角**RecyclerView**比较特殊，具有**ViewHolder**的概念，那么我们也可以把它定位组件，而**RecyclerView**就是整体。这些组件可以在任何地方用，组成不同的列表效果，组成规则静态写死也好，服务端动态控制也好只不过是一个模板规则。

![](/img/image_bu_ming_zhen_xiang.jpg)

### 实现
实现之前我们一定要有一个思路，这样能事半功倍。假设我们要把**ViewHolder**定义为组件，那么它就应该像四大组件**Activity**一样独立，比如像这样子：

```
@BindLayout(layoutId = R.layout.item_type_one)
public class TypeOneViewHolder extends BaseViewHolder<MultiTypeEntity> {
    ...
    @Override
    protected void showData(MultiTypeEntity data, int position) {
        super.showData(data, position);
        ((TextView) itemView).setText("type:" + data.getItemType() + " isVisible:" + mSubject.isVisible());
    }
    ...
}
```

每个**ViewHolder**只要根据数据渲染对应UI就行了，不需要考虑很多。

#### 注册
从上面的例子，我们了解到，每个**ViewHolder**可以通过注解绑定一个layoutId，就跟**Activity**的**setContentView**一样，直接在组件中暴露布局文件其实有利有弊，好处就是修改的时候我可以直接点进去，效率很高；但正因为这种绑定导致布局文件的定死，不可随意修改，现在越来越多这样的场景：布局元素一样，但是各元素大小可能不一样，位置可能不一样，但UI逻辑是一样的。针对这两种情况我会给出对应的策略。

例如第一种，我们可以采用注解绑定，就如我们看到的例子一样；第二种，我们可以借助**MTRVA**的资源管理，例如像这样：

```
// 这里需要一个注意点，如果你采用的MTRVA的标准模式，也就是带level的模式，同时又是通过注解绑定就需要空注册，非注解而是下面这种无需空注册
mHelper.registerModule(0)
        .type(TypeOne.TYPE)
        .layoutResId(R.layout.item_type_one)
        .register();
        
// 空注册
mHelper.registerModule(Integer.MIN_VALUE).register();
```

OK，上面就是我们的布局文件注册，那么我们的组件是如何注册的呢？如何跟**Adapter**关联的呢？

```
@NonNull
@Override
public final CrazyViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
    mContext = parent.getContext();
    LayoutInflater inflater = LayoutInflater.from(mContext);
    // 在这，我们获取到组件的class对象
    Class<? extends CrazyViewHolder> vhClass = mResourceManager.get(viewType);
    // 通过注解获取布局文件
    BindLayout layoutAnnotation = vhClass.getAnnotation(BindLayout.class);
    int layoutId;
    if (layoutAnnotation == null) {
        // 如果拿不到注解就会MTRVA去拿
        layoutId = mHelper.getLayoutId(viewType);
    } else {
        layoutId = layoutAnnotation.layoutId();
    }
    if (layoutId == 0) {
        // 如果都拿不到，就会抛出异常
        throw new RuntimeException("请在子组件中使用BindLayout注解绑定layoutId或者通过AdapterHelper注册layoutId");
    }
    // 通过布局文件id创建view
    View view = getItemView(inflater, parent, layoutId);
    // 通过反射创建组件
    CrazyViewHolder viewHolder = createCrazyViewHolder(vhClass, view);
    CrazyViewHolder vh = viewHolder == null ? new CrazyViewHolder(view) : viewHolder;
    // 子Adapter可以对组件特殊处理
    onViewHolderCreated(vh);
    return vh;
}
```

从上面可知我们是通过**mResourceManager**拿到组件的class对象，那么怎么放进去的，闭着眼睛都知道。

```
// 多用于动态注册，key为itemType
public final void register(int itemType, Class<? extends CrazyViewHolder> vhClass) {
    mResourceManager.put(itemType, vhClass);
}
// 例子
register(TypeOne.TYPE, TypeOneViewHolder.class);

// 自己创建一个map集合传入
public CrazyAdapter(@NonNull RecyclerViewAdapterHelper<T> helper, @NonNull SparseArray<Class<? extends CrazyViewHolder>> manager) {
    ...
}

```

到这里，一般的纯展示效果可以轻松搞定。如果现实也能这么简单，那该多好啊！

![](/img/image_laoshi_bajiao.jpg)

#### 通信
每个组件总是避免不了和他人通信，总是避免不了命中注定。为什么这么熟悉呢？可能是喝多了吧！命中注定就是我们组件的生命周期，组件的一生逃不掉出生、死亡、出现和消失，多么地无奈啊！

```
// 出生
public CrazyViewHolder(@NonNull View itemView) {
    super(itemView);
}
// 出现
protected void onViewAttachedToWindow() {
}
// 消失
protected void onViewDetachedFromWindow() {
}
// 死亡，开始轮回
protected void onViewRecycled() {
}
```

组件在这个大千世界里总是避免不了工作，朝九晚六亦或是九九六。这都是你的大老板（**Activity**或者**Fragment**等）安排，连你的组长**RecyclerView**也得听着。那么工作之前，你得面试，面试造火箭，工作拧螺丝大家都知道吧。

![](/img/image_xiaozhang_shanshanzi.gif)

我们来看看如何造火箭，如何拧螺丝吧！这里我以打造一个上拉加载更多为例：

```
组件和主容器通信需要一个桥梁，这里提供一个接口，一般为Adapter实现该接口
public interface ComponentSubject {

    // 获取主容器的上拉加载状态
    int loadMoreState();

    // 除了拉模式，还有推模式，但是推模式最好需要一个响应的过程，这里提供标识接口IEvent
    // 标识接口你可以实现各种各样的功能，除了上拉加载，你也可以实现主容器的生命周期监听等等
    void notifyEvent(@NonNull Object event, @Nullable IEvent callback);
}
```

这里以**Adapter**实现该接口：

```
@Override
public void notifyEvent(@NonNull Object event, @Nullable IEvent iEvent) {
    // 可以把该事件通知给主容器
    if (mEventCallback != null) {
        mEventCallback.onEvent(event, iEvent);
    }
    // 当然也可以把该事件由Adapter处理，主容器和组件通信需要Adapter协调，起到封装隔离的作用，具体看你的业务场景，没有具体的架构
    if (event instanceof LoadMore && iEvent instanceof LoadMoreEvent) {
        // 一切的上拉加载真正的逻辑处理全由LoadMoreHelper处理
        mLoadMoreHelper.handleLoadMoreEvent((LoadMore) event, (LoadMoreEvent) iEvent);
    }
}
```

为什么提供**LoadMoreHelper**？很明显，职责分明，代码更加清晰，易于维护，如果需要监听主容器的生命周期就可以加一个**LifeCycleHelper**，哈哈。我们来看看，**LoadMoreHelper**到底干了什么事。

```
public class LoadMoreHelper {
    ...
    // 开启上拉加载
    public void setLoadMoreEnable(boolean loadMoreEnable) {
        ...
    }
    // 加载状态
    public int loadMoreState() {
        return loadMoreState;
    }
    // 隐藏上拉加载
    public void loadMoreGone() {
        ...
    }
    // 开始加载
    private void loadMoreLoading() {
        ...
    }
    // 加载完成
    public void loadMoreCompleted() {
        ...
    }
    // 加载失败
    public void loadMoreFailed() {
        ...
    }
    // 加载结束
    public void loadMoreEnd() {
        loadMoreState = LoadMoreViewHolder.STATE_END;
        if (loadMoreEvents == null) {
            return;
        }
        for (LoadMoreEvent event : loadMoreEvents) {
            event.onEnd();
        }
    }
    // 默认加载
    public void loadMoreDefault() {
        ...
    }
    // 处理一些组件的通知时间，例如上拉加载初始化，开始加载等等
    public void handleLoadMoreEvent(@NonNull LoadMore loadMore, @NonNull LoadMoreEvent event) {
        if (loadMore.state == LoadMore.STATE_INIT) {
            if (loadMoreEvents == null) {
                loadMoreEvents = new HashSet<>();
            }
            loadMoreEvents.add(event);
            return;
        }
        ...
    }
    ...
}
```

大致逻辑也很简单，组件发通知过来，把事件回调引用存下来，一般是**ViewHolder**自身，例如客户端已经是拿到分页的最后一页了，需要通知组件加载已经结束，通过遍历通知所有的观察者，是的，这是一种`观察者模式`，无处不在。此时，组件就会收到来自主容器的问候，很开心，很感动。

![](/img/image_nuli_gongzuo.jpeg)

```
public class LoadMoreViewHolder extends BaseViewHolder implements LoadMoreEvent{
	...
	@Override
    public void onEnd() {
        // 组件处理加载结束的UI逻辑
        ...
    }
    ...
}

public interface LoadMoreEvent extends IEvent {
    // 默认
    void onDefault();
    // 隐藏
    void onGone();
    // 加载
    void onLoading();
    // 失败
    void onFailed();
    // 完成
    void onCompleted();
    // 结束
    void onEnd();
}
```

可以看到一次完整的请求响应还是挺麻烦的，但是大型的项目，好的项目，优秀的规范是必要的，邮件的层层通知也是有必要的，不像小公司很自由，但最后只会不断地填坑，不断地填坑，不断地填坑直至无法再填，OK，需求取消。那么小公司的代码怎么做呢？**EventBus**啊，是不是啊？你项目的**EventBus**维护可还好？

### 集成
为了你们，正在九九六的我挤出了一点时间给你们封装了一个组件库，当然这个库是依赖**MTRVA**的。如果你有什么更好的封装思路请联系我。如果按以上思路下来，每次你创建一个列表的时候，只要创建对应组件就行了，连**Adapter**都不用写，如果有复用的组件，啥都不用写了，是不是很舒服？集成方式也很简单，**MTRVA**是必须的，如果你要用到组件库，那么就依赖组件库，如果不需要想自己封装，那么就无需集成，两个库是独立的。

![](/img/image_bangbangde.jpeg)

## 结束语
看到这里的同学，我相信你一定收获了不少表情包，额，不是，收获了不少新知识。话说回来，确实好久没写文章了，但自己还是时刻`关注`技术，`热爱`技术的，即使它显得没那么重要。这次的组件化思路我很早就提出来了，当然也是受别人影响，中间踩过很多坑，也换了很多模式设计。当时推出**MTRVA**的时候没有一并带出，是因为我觉得这并不是很难的东西，只是业务场景不一样，但还是很多人跟我的demo一样，一堆逻辑写在**onBindViewHolder**，难道是我的demo害人了？后续我会把开源项目[CrazyDaily](https://github.com/crazysunj/CrazyDaily)里面也改成组件化，有兴趣的同学可以关注下，当然不管项目还是库也好都会持续更新。
>可能会迟到，但永远不会缺席，除非我不再做技术。

![](/img/image_gandong_ziji.jpeg)


## 传送门
Github：[https://github.com/crazysunj/](https://github.com/crazysunj/)


