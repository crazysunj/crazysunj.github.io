---
title: 走进RecyclerView
toc: true
date: 2018-11-23 10:36:03
tags: [RecyclerView]
---
## 前言
好久没写文章，今天看了眼RecyclerView源码，水一篇吧！毕竟我还是非常热爱技术的。

![](/img/jiexialai.jpg)

这次分析RecyclerView主要从绘制和缓存出发，来吧，朋友！
<!--  more-->
## 绘制
### onMeasure
```
@VisibleForTesting LayoutManager mLayout;
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    ...
 }
```
这是第一部分的测量，如果我们的LayoutManager为null，那么就不能交给LayoutManager来测量，只好自己测量喽！测量逻辑也很简单：

```
void defaultOnMeasure(int widthSpec, int heightSpec) {
    final int width = LayoutManager.chooseSize(widthSpec,
            getPaddingLeft() + getPaddingRight(),
            ViewCompat.getMinimumWidth(this));
    final int height = LayoutManager.chooseSize(heightSpec,
            getPaddingTop() + getPaddingBottom(),
            ViewCompat.getMinimumHeight(this));
    // 设置对应的宽高
    setMeasuredDimension(width, height);
}

public static int chooseSize(int spec, int desired, int min) {
    final int mode = View.MeasureSpec.getMode(spec);
    final int size = View.MeasureSpec.getSize(spec);
    switch (mode) {
        case View.MeasureSpec.EXACTLY:
            return size;
        case View.MeasureSpec.AT_MOST:
            // 取padding和最小宽(高)的最大值，再取其与size的最小值
            // 常规操作
            return Math.min(size, Math.max(desired, min));
        case View.MeasureSpec.UNSPECIFIED:
        default:
            return Math.max(desired, min);
    }
}
```

这是第二部分的代码（else部分直接过掉）：

```
// 姑且翻译成自动测量吧，LayoutManager默认是false，但系统提供的几个都是重写isAutoMeasureEnable返回ture，例如LinearLayoutManager。
 if (mLayout.isAutoMeasureEnabled()) {
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);
    // 这里走的一般都是LayoutManager基类的onMeasure，至少系统提供的貌似都没重写，其底层调用的是RecyclerView的defaultOnMeasure
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

    final boolean measureSpecModeIsExactly =
            widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
    // 很明显，如果是EXACTLY模式已经够用了
    if (measureSpecModeIsExactly || mAdapter == null) {
        return;
    }
    ...
}
```

第三部分的代码：

```
// 开始环节，State分为开始环节，布局环节和动画环节
if (mState.mLayoutStep == State.STEP_START) {
    // 主要用于保存一些view的信息，然后进入布局环节
    dispatchLayoutStep1();
}
// 赋值操作
mLayout.setMeasureSpecs(widthSpec, heightSpec);
mState.mIsMeasuring = true;
// 开始对child进行布局，主要由相应的LayoutManager自己去实现，然后进入动画环节
dispatchLayoutStep2();
// child布局完就可以根据child的宽高进行最终的布局测量
mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
```

第四部分：

```
// 默认为false，相应LayoutManager可以实现相应shouldMeasureTwice进行重写修改
if (mLayout.shouldMeasureTwice()) {
    mLayout.setMeasureSpecs(
            MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
            MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
    mState.mIsMeasuring = true;
    // 主要针对child的重新布局，重新测量一遍
    dispatchLayoutStep2();
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
}
```

### onLayout

```
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}

void dispatchLayout() {
    ...
    // 主要走dispatchLayoutStep3，前面代码主要保证是否已经走完测量和布局
    // 还记得dispatchLayoutStep1吗？用于保存一些信息？什么信息？测量布局之前的信息。dispatchLayoutStep3干什么？保存测量布局之前的信息。OK，两者都有的情况下，就可以执行动画啦！
    dispatchLayoutStep3();
}
```

## 缓存
这里提供主要的逻辑线索帮助你。还记得dispatchLayoutStep2吗？这里只是简单告诉你用于child布局，既然是child布局，那么必然要先获取child，我们这里以LinearLayoutManager为例。

```
// LinearLayoutManager
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
	...
	// 不断根据锚点填充view
	fill(recycler, mLayoutState, state, false);
	...
}

int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
	...
	layoutChunk(recycler, state, layoutState, layoutChunkResult);
	...
}

void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
	...
	View view = layoutState.next(recycler);
	...
	addView(view);
	...
	addDisappearingView(view);
	...
}

// LayoutState
View next(RecyclerView.Recycler recycler) {
    ...
    final View view = recycler.getViewForPosition(mCurrentPosition);
    ...
    return view;
}

// RecyclerView.Recycler
@NonNull
public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}
```

介绍关键代码之前，我们先了解一下相关数据结构！

```
// view被detach并非remove，同时又是有效的或者已经remove的，会被存储在该集合
final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
// view被detach并非remove，但既不是有效的也不是已经remove的
ArrayList<ViewHolder> mChangedScrap = null;
// 缓存满足isRecyclable的view
final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
// mAttachedScrap的不可修改集合
private final List<ViewHolder>
        mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);
// 共享缓存池
RecycledViewPool mRecyclerPool;
// 自定义的一个view池，同时自己去维护
private ViewCacheExtension mViewCacheExtension;
```

OK，我们再来看看这些数据结构是如何管理我们的view的，我们先从“取”开始，承接上面的线索tryGetViewHolderForPositionByDeadline：

```
@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
	...
	ViewHolder holder = null;
	...
	// 跟进后的代码，先从mChangedScrap取
	holder = mChangedScrap.get(i);
	...
	if (holder == null) {
		...
		// 通过position分别从mAttachedScrap和mCachedViews中取
		holder = mAttachedScrap.get(i);
		if (holder == null) {
			holder = mCachedViews.get(i);
		}
		...
	}
	if (holder == null) {
		...
		// 通过id分别从mAttachedScrap和mCachedViews中取
		holder = mAttachedScrap.get(i);
		if (holder == null) {
			holder = mCachedViews.get(i);
		}
		...
		if (holder == null && mViewCacheExtension != null) {
			...
			// 从mViewCacheExtension中取
			final View view = mViewCacheExtension
                            .getViewForPositionAndType(this, position, type);
			...
			holder = getChildViewHolder(view);
			...
		}
		if (holder == null) {
			...
			// 从mRecyclerPool中取
			holder = getRecycledViewPool().getRecycledView(type);
			...
		}
		if (holder == null) {
			...
			// 都没找到，通过Adapter去创建
			holder = mAdapter.createViewHolder(RecyclerView.this, type);
			...
		}
	}
	...
	return holder;
}
```

说完“取”，我们再来说说“存”。挑选“存”的地方，我挑在重新布局的地方，因为重新布局首先肯定先移除child，再添加child。我们还是以LinearLayoutManager为例。

```
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
	// 有这么一句话
	...
	detachAndScrapAttachedViews(recycler);
	...
}

public void detachAndScrapAttachedViews(@NonNull Recycler recycler) {
    ...
    scrapOrRecycleView(recycler, i, v);
    ...
}

private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    ...
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        ...
        // 这里用于mCachedViews和mRecyclerPool的存
        // 先存mCachedViews，如果没满足条件才会存进mRecyclerPool
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        ...
        // 这里用于mAttachedScrap和mChangedScrap
        // mAttachedScrap存有效或者已经移除
        recycler.scrapView(view);
        ...
    }
}
```

这里并没有mViewCacheExtension，是的，一开始就说了，这玩意，自己玩去吧。
## 结束语
好吧，差不多水完了，本篇差不多是对RecyclerView一个大体的了解，实际遇到问题，一般毫无卵用，还是要去看源码的实现细节。加油吧，小伙伴！这只是走进，自古走进简单，走出难！
