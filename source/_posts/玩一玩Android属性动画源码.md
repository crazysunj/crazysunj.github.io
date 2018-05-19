---
title: 玩一玩Android属性动画源码
toc: true
date: 2018-05-19 17:55:42
tags: [Android,Animator,Choreographer]
---

### 前言
主要是对[来一份Android动画全家桶]()转场动画章节中属性动画的源码分析，辅助读者更好了解数据的流向。

**_本文基于sdk27_**

**_本文特别特别特别长_**

**_耐心稍欠同学可以直接看流程图_**
### 分析
万恶之源：

```
ObjectAnimator.ofObject(view, new PropPosition(PointF.class, "position", new PointF(endRect.centerX(), endRect.centerY())), null, path);
//ObjectAnimator
@NonNull
public static <T, V> ObjectAnimator ofObject(T target, @NonNull Property<T, V> property,
        @Nullable TypeConverter<PointF, V> converter, Path path) {
    PropertyValuesHolder pvh = PropertyValuesHolder.ofObject(property, converter, path);
    return ofPropertyValuesHolder(target, pvh);
}
```
<!--  more-->
根据参数跟进PropertyValuesHolder的静态方法ofObject：

```
//PropertyValuesHolder
public static <V> PropertyValuesHolder ofObject(Property<?, V> property,
            TypeConverter<PointF, V> converter, Path path) {
    PropertyValuesHolder pvh = new PropertyValuesHolder(property);
    pvh.mKeyframes = KeyframeSet.ofPath(path);
    pvh.mValueType = PointF.class;
    pvh.setConverter(converter);
    return pvh;
}
```

继续根据参数跟进：

```
//PropertyValuesHolder
private PropertyValuesHolder(Property property) {
    mProperty = property;
    if (property != null) {
        mPropertyName = property.getName();
    }
}

public void setConverter(TypeConverter converter) {
    mConverter = converter;
}
```

这里，我们整理一下PropertyValuesHolder存储的变量，

* mProperty	->	传进去的Property
* mPropertyName	->	传进去的Property的name
* mConverter	->	传进去的TypeConverter
* mKeyframes	->	根据传进去的path转换的Keyframes对象
* mValueType	->	PointF.class

回到ObjectAnimator的ofObject方法，处理完PropertyValuesHolder的初始化，再来看看ObjectAnimator对PropertyValuesHolder后续操作：

```
//ObjectAnimator
@NonNull
public static ObjectAnimator ofPropertyValuesHolder(Object target,
        PropertyValuesHolder... values) {
    ObjectAnimator anim = new ObjectAnimator();
    anim.setTarget(target);
    anim.setValues(values);
    return anim;
}
```

从ofObject方法可知values就是我们传入的PropertyValuesHolder，因此继续跟进setValues：

```
//ObjectAnimator
public void setValues(PropertyValuesHolder... values) {
    int numValues = values.length;
    mValues = values;
    mValuesMap = new HashMap<String, PropertyValuesHolder>(numValues);
    for (int i = 0; i < numValues; ++i) {
        PropertyValuesHolder valuesHolder = values[i];
        mValuesMap.put(valuesHolder.getPropertyName(), valuesHolder);
    }
    // New property/values/target should cause re-initialization prior to starting
    mInitialized = false;
}
```
整理一下ObjectAnimator的变量，

* mValues	->	我们根据传入参数构建的PropertyValuesHolder对象
* mValuesMap	->	key为PropertyValuesHolder的mPropertyName，value为PropertyValuesHolder本身

貌似到这里就断了，那是因为以上只是初始化变量，只有动画执行的时候，才会去操作变量。

OK，动画的开始方法是start，那么我们跟进去瞧瞧：

```
//ObjectAnimator
@Override
public void start() {
    AnimationHandler.getInstance().autoCancelBasedOn(this);
    if (DBG) {
        ...
    }
    super.start();
}
```

OK，看不出什么花样，跟进super看看：

```
//ValueAnimator
@Override
public void start() {
    start(false);
}

private void start(boolean playBackwards) {
    ...
    addAnimationCallback(0);
    if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
        ...
        startAnimation();
        if (mSeekFraction == -1) {
            ...
            setCurrentPlayTime(0);
        } else {
            setCurrentFraction(mSeekFraction);
        }
    }
}
```
最下面的setCurrentPlayTime和setCurrentFraction设置进度，并初始化到设置进度，过滤掉。

看名字肯定走进startAnimation去瞧瞧，那就走吧：

```
//ValueAnimator
private void startAnimation() {
    ...
    initAnimation();
    ...
    if (mListeners != null) {
        notifyStartListeners();
    }
}
```
看initAnimation这个名字应该是变量的初始化，但还是忍不住进去看看：

```
//ValueAnimator
@CallSuper
void initAnimation() {
    if (!mInitialized) {
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].init();
        }
        mInitialized = true;
    }
}
```

还记得我们的变量总结吗？mValues就是我们根据传进去的参数构建的PropertyValuesHolder对象。进去瞧瞧，初始化了啥：

```
//PropertyValuesHolder
void init() {
    if (mEvaluator == null) {
     	...
        mEvaluator = (mValueType == Integer.class) ? sIntEvaluator :
                (mValueType == Float.class) ? sFloatEvaluator :
                null;
    }
    if (mEvaluator != null) {
        ...
        mKeyframes.setEvaluator(mEvaluator);
    }
}
```
好像毫无卵用，这里解释下mEvaluator，这玩意叫估值器，还记得ArgbEvaluator吗？说白了就是用来计算动画属性的。

过，过，过，再回到ValueAnimator的startAnimation方法，进去notifyStartListeners方法瞧瞧，看见notify字眼，说不定就是：

```
//ValueAnimator
private void notifyStartListeners() {
    if (mListeners != null && !mStartListenersCalled) {
        ArrayList<AnimatorListener> tmpListeners =
                (ArrayList<AnimatorListener>) mListeners.clone();
        int numListeners = tmpListeners.size();
        for (int i = 0; i < numListeners; ++i) {
            tmpListeners.get(i).onAnimationStart(this, mReversing);
        }
    }
    mStartListenersCalled = true;
}
```

直接回调了onAnimationStart方法，好像不太对，不过我还是想知道clone的mListeners是什么东西，找找：

```
//Animator
public void addListener(AnimatorListener listener) {
    if (mListeners == null) {
        mListeners = new ArrayList<AnimatorListener>();
    }
    mListeners.add(listener);
}
```
搜得死内，就是我们设置的监听，但不是重点，我们现在还不知道在哪里回调数据。再回去，回到ValueAnimator的start(boolean playBackwards)方法，只剩下addAnimationCallback方法了，应该是吧？瞧瞧？

```
//ValueAnimator
private void addAnimationCallback(long delay) {
    if (!mSelfPulse) {
        return;
    }
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}
```

what？getAnimationHandler是什么东西？不会是Handler吧？了解一下？

```
//ValueAnimator
public AnimationHandler getAnimationHandler() {
    return AnimationHandler.getInstance();
}
```
哦靠，白进了，继续跟进吧！

```
/**
 * This custom, static handler handles the timing pulse that is shared by all active
 * ValueAnimators. This approach ensures that the setting of animation values will happen on the
 * same thread that animations start on, and that all animations will share the same times for
 * calculating their values, which makes synchronizing animations possible.
 *
 * The handler uses the Choreographer by default for doing periodic callbacks. A custom
 * AnimationFrameCallbackProvider can be set on the handler to provide timing pulse that
 * may be independent of UI frame update. This could be useful in testing.
 *
 * @hide
 */
public class AnimationHandler {
	...
}
```
大吃一斤，竟然不是Handler，简单翻译一下AnimationHandler的简介：

>自定义的静态Handler处理被所有活跃的ValueAnimator共享的定时的脉冲。这可以确保动画开始后发生的动画值在同一线程，并且所有的动画将共享相同的时间来计算它们的值，这样可以使动画同步。

>默认情况下，Handler使用Choreographer处理定期回调，自定义的AnimationFrameCallbackProvider设置在Handler为了提供独立于UI帧更新的定时的脉冲。这在测试中是有用的。

自定义的静态Handler？我这暴脾气，疯狂搜索源码，发现：4.1.1以前AnimationHandler的确是继承Handler，其后一段时间还有实现Runnable的，好吧，不管它了，应该是没修改注释。虽然用蹩脚地翻译了一下，但是并没卵用，因为我们还不知道它说的是什么，可能第一段的信息比较给力，就是AnimationHandler可以保证动画的同步。

OK，我们回到：

```
//ValueAnimator
private void addAnimationCallback(long delay) {
    if (!mSelfPulse) {
        return;
    }
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}
```
这说ValueAnimator 实现了AnimationHandler.AnimationFrameCallback，果不其然：

```
public class ValueAnimator extends Animator implements AnimationHandler.AnimationFrameCallback
```
继续走进addAnimationFrameCallback方法：

```
//AnimationHandler
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
        getProvider().postFrameCallback(mFrameCallback);
    }
    if (!mAnimationCallbacks.contains(callback)) {
        mAnimationCallbacks.add(callback);
    }

    if (delay > 0) {
        mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
    }
}
```

这里有几个关键变量，我们先记一下：

* mAnimationCallbacks
* getProvider()
* mFrameCallback
* mDelayedCallbackStartTime

第二个是个方法，先去看看是什么变量：

```
//AnimationHandler
private AnimationFrameCallbackProvider getProvider() {
    if (mProvider == null) {
        mProvider = new MyFrameCallbackProvider();
    }
    return mProvider;
}
```
确定是mProvider，往上拉，看变量，有木有注释：

```
//AnimationHandler
/**
 * Internal per-thread collections used to avoid set collisions as animations start and end
 * while being processed.
 * @hide
 */
private final ArrayMap<AnimationFrameCallback, Long> mDelayedCallbackStartTime =
        new ArrayMap<>();
private final ArrayList<AnimationFrameCallback> mAnimationCallbacks =
        new ArrayList<>();
private final ArrayList<AnimationFrameCallback> mCommitCallbacks =
        new ArrayList<>();
private AnimationFrameCallbackProvider mProvider;

private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
            getProvider().postFrameCallback(this);
        }
    }
};
```

从注释以及命名上看mDelayedCallbackStartTime应该是根据key为动画，value为开始延迟时间存储。mAnimationCallbacks就是存储我们的动画。那么关键点就是这句话了：

```
//AnimationHandler
if (mAnimationCallbacks.size() == 0) {
    getProvider().postFrameCallback(mFrameCallback);
}
```
mFrameCallback我们先大胆地猜测是AnimationHandler注释里面说的Choreographer回调。

先去看看它有没有注释，注释不就是帮助我们分析问题的吗？

```
//Choreographer
/**
 * Implement this interface to receive a callback when a new display frame is
 * being rendered.  The callback is invoked on the {@link Looper} thread to
 * which the {@link Choreographer} is attached.
 */
public interface FrameCallback {
    public void doFrame(long frameTimeNanos);
}
```

我们猜的没错，每次渲染的时候都会通过这个接口回调。

OK，只差最后一个变量了mProvider，老套路，看注释：

```
//AnimationHandler
/**
 * The intention for having this interface is to increase the testability of ValueAnimator.
 * Specifically, we can have a custom implementation of the interface below and provide
 * timing pulse without using Choreographer. That way we could use any arbitrary interval for
 * our timing pulse in the tests.
 *
 * @hide
 */
public interface AnimationFrameCallbackProvider {
    void postFrameCallback(Choreographer.FrameCallback callback);
    void postCommitCallback(Runnable runnable);
    long getFrameTime();
    long getFrameDelay();
    void setFrameDelay(long delay);
}
```

好像没什么卵用？有一句好像很关键，实现这个接口是不想直接用Choreographer。先记住，既然是接口，必然有实现类，找一找：

```
//AnimationHandler
/**
 * Default provider of timing pulse that uses Choreographer for frame callbacks.
 */
private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {
    final Choreographer mChoreographer = Choreographer.getInstance();
    @Override
    public void postFrameCallback(Choreographer.FrameCallback callback) {
        mChoreographer.postFrameCallback(callback);
    }

    @Override
    public void postCommitCallback(Runnable runnable) {
        mChoreographer.postCallback(Choreographer.CALLBACK_COMMIT, runnable, null);
    }

    @Override
    public long getFrameTime() {
        return mChoreographer.getFrameTime();
    }

    @Override
    public long getFrameDelay() {
        return Choreographer.getFrameDelay();
    }

    @Override
    public void setFrameDelay(long delay) {
        Choreographer.setFrameDelay(delay);
    }
}
```

果然跟注释说的一样，AnimationFrameCallbackProvider就是个包装类。

OK，到这里，我们先整理一下当初提出的几个变量：

* mAnimationCallbacks：用来存储我们的动画示例
* mProvider：用来包装Choreographer
* mFrameCallback：每次渲染时候的回调接口
* mDelayedCallbackStartTime：存储每个动画开始的延迟时间

继续我们的分析，那么所有的线索都指向Choreographer，例如mFrameCallback的接口也是Choreographer里面的，看起来是到核心类了，进去玩一玩：

```
//Choreographer
/**
 * Coordinates the timing of animations, input and drawing.
 * <p>
 * The choreographer receives timing pulses (such as vertical synchronization)
 * from the display subsystem then schedules work to occur as part of rendering
 * the next display frame.
 ...
 */
public final class Choreographer {
}
```
注释很长，我们就挑了关键的，这个类是用来处理动画、输入和绘制。Choreographer会接受显示子系统的定时脉冲然后在下一帧处理渲染工作。

貌似是个重量级类，跟系统打交道，那八成有Binder通信，好像分析才刚刚开始。来吧！类虽然只有1000行不到，但我们还是从线索开始分析，这样更好分析，也更好理解。

回到我们的卡点：

```
//AnimationHandler
if (mAnimationCallbacks.size() == 0) {
    getProvider().postFrameCallback(mFrameCallback);
}
```
从我们前面分析的，mProvider只是个包装类，内部其实是Choreographer，那么我们直接看Choreographer的postFrameCallback方法：

```
//Choreographer
public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        ...
    }
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
    ...
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

记录一下参数：

* callbackType：CALLBACK\_ANIMATION
* action：callback
* token：FRAME\_CALLBACK\_TOKEN
* delayMillis：0

else部分只是延迟执行，最后还是会走scheduleFrameLocked方法。我们先了解一下mCallbackQueues是什么东西。

```
//Choreographer
private final CallbackQueue[] mCallbackQueues;
private final class CallbackQueue {
    private CallbackRecord mHead;
    ...
}

private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}
```

mCallbackQueues是CallbackQueue的数组，而CallbackQueue是管理CallbackRecord的一个队列，addCallbackLocked方法就是把参数分装成CallbackRecord对象添加到队列中，根据我们刚才整理的变量，会走doFrame方法，先记着。

再来看看scheduleFrameLocked方法：

```
//Choreographer
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            ...
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            ...
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

分为两个分支，一个是使用垂直同步，一个是不使用。垂直同步很简单，就是让你的渲染频率和屏幕刷新频率保持同步。这里我们直接过掉不必要的步骤，如果是垂直同步，最终会调这个，只是如果是当前的loop线程会直接执行，不然就利用handler异步执行：

```
//Choreographer/FrameHandler
case MSG_DO_SCHEDULE_VSYNC:
    doScheduleVsync();
    break;
//Choreographer
void doScheduleVsync() {
    synchronized (mLock) {
        if (mFrameScheduled) {
            scheduleVsyncLocked();
        }
    }
}
private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}
//DisplayEventReceiver
public void scheduleVsync() {
    ...
    nativeScheduleVsync(mReceiverPtr);
    ...
}
private static native void nativeScheduleVsync(long receiverPtr);
```
既然已经到native了，我们先放放。还剩下一个是非垂直同步的，Handler发送消息执行：

```
//Choreographer$FrameHandler
case MSG_DO_FRAME:
    doFrame(System.nanoTime(), 0);
    
//Choreographer
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
    synchronized (mLock) {
        ...
        if (frameTimeNanos < mLastFrameTimeNanos) {
            ...
            //处理跳帧
            scheduleVsyncLocked();
            return;
        }
        ...
    }
    ...
    //我们这里看处理动画的，其实都会走doCallbacks方法
    mFrameInfo.markAnimationsStart();
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
	...
}
```

那么现在只剩下doCallbacks了，一起来看看：

```
//Choreographer
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        ...
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        ...
    }
    try {
        ...
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            ...
            c.run(frameTimeNanos);
        }
    } finally {
    	//回收callbacks
        synchronized (mLock) {
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
   		...
    }
}
```

首先callbacks赋值，mCallbackQueues是不是很熟悉，还记得这句话吗？

```
//Choreographer
mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
```

不过当时addCallbackLocked的代码没贴出来，现在我们和extractDueCallbacksLocked一起过下：

```
//Choreographer/CallbackQueue
public void addCallbackLocked(long dueTime, Object action, Object token) {
    CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
    CallbackRecord entry = mHead;
    if (entry == null) {
        mHead = callback;
        return;
    }
    if (dueTime < entry.dueTime) {
        callback.next = entry;
        mHead = callback;
        return;
    }
    while (entry.next != null) {
        if (dueTime < entry.next.dueTime) {
            callback.next = entry.next;
            break;
        }
        entry = entry.next;
    }
    entry.next = callback;
}

public CallbackRecord extractDueCallbacksLocked(long now) {
    CallbackRecord callbacks = mHead;
    if (callbacks == null || callbacks.dueTime > now) {
        return null;
    }

    CallbackRecord last = callbacks;
    CallbackRecord next = last.next;
    while (next != null) {
        if (next.dueTime > now) {
            last.next = null;
            break;
        }
        last = next;
        next = next.next;
    }
    mHead = next;
    return callbacks;
}
```

稍微有点绕，仔细整理下，addCallbackLocked就是把新生产CallbackRecord添加到末尾，extractDueCallbacksLocked就是提取里面的首个CallbackRecord，相当于全部，提取完得清空。

现在再回来看这段代码：

```
//Choreographer
void doCallbacks(int callbackType, long frameTimeNanos) {
	...
	for (CallbackRecord c = callbacks; c != null; c = c.next) {
        ...
        c.run(frameTimeNanos);
    }
    ...
}
```

OK，很简单，就是执行所有CallbackRecord的run方法，还记得我让你们记住的run方法吗？我猜可能忘了，再放一遍：

```
//Choreographer/CallbackRecord
public void run(long frameTimeNanos) {
    if (token == FRAME_CALLBACK_TOKEN) {
        ((FrameCallback)action).doFrame(frameTimeNanos);
    } else {
        ((Runnable)action).run();
    }
}
```

根据我们当初的传参会执行doFrame方法，这个总记得吧！根据我们当初对变量的分析，action就是我们AnimationHandler里面的mFrameCallback。

这句话应该没忘吧？

```
//AnimationHandler
if (mAnimationCallbacks.size() == 0) {
    getProvider().postFrameCallback(mFrameCallback);
}
```

然后再来解释为啥这里等于0的时候，发了个信号，因为这是异步，同时也是发起点。OK，我们再来仔细研究下mFrameCallback这个变量，虽然上面已经贴出过：

```
//AnimationHandler
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
    		//再发下一个信号
            getProvider().postFrameCallback(this);
        }
    }
};
```

是不是很激动，好像快要结束了？来吧。

```
//AnimationHandler
private void doAnimationFrame(long frameTime) {
    ...
    for (int i = 0; i < size; i++) {
        final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
        ...
        callback.doAnimationFrame(frameTime);
        ...
    }
	cleanUpList();
}
```

虽然给你们总结了各种变量，但我觉得你们还是忘了，回顾一下mAnimationCallbacks出现地点：

```
//AnimationHandler
if (!mAnimationCallbacks.contains(callback)) {
    mAnimationCallbacks.add(callback);
}
```
记起来了吧，存储我们的动画实例，因为我们的ValueAnimator实现了AnimationHandler.AnimationFrameCallback。那么我们去ValueAnimator里面看看doAnimationFrame方法。

```
//ValueAnimator
public final boolean doAnimationFrame(long frameTime) {
    ...
    mLastFrameTime = frameTime;
    ...
    boolean finished = animateBasedOnTime(currentTime);

    if (finished) {
        endAnimation();
    }
    return finished;
}
```

原谅我疯狂的省略，因为本文已经很长了，这里重点是animateBasedOnTime方法，更进：

```
//ValueAnimator
boolean animateBasedOnTime(long currentTime) {
    boolean done = false;
    if (mRunning) {
        ...
        animateValue(currentIterationFraction);
    }
    return done;
}

@CallSuper
void animateValue(float fraction) {
    fraction = mInterpolator.getInterpolation(fraction);
    mCurrentFraction = fraction;
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].calculateValue(fraction);
    }
    if (mUpdateListeners != null) {
        int numListeners = mUpdateListeners.size();
        for (int i = 0; i < numListeners; ++i) {
            mUpdateListeners.get(i).onAnimationUpdate(this);
        }
    }
}
```

敲黑板，重点！重点！重点！还记得我们为了哪几个数据的走向，写了这篇文章吗？

重点是mValues，我又想说我给你们整理的变量，mValues就是我们传进去的PropertyValuesHolder对象，而它执行了calculateValue方法，去看看：

```
//PropertyValuesHolder
void calculateValue(float fraction) {
    Object value = mKeyframes.getValue(fraction);
    mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
}
```

我给你们...整理..的变...算了，不说了，mKeyframes是根据传进去的path转换的Keyframes对象，mConverter是传进去的TypeConverter，我们传的是null，这也是为啥我说这玩意就是个类型转换器，我们用的PonitF，而默认的也是PointF。然后赋值给mAnimatedValue。

好像还没结束，到这里只是数值计算结束了，诶，不对，为啥我们看的是ValueAnimator的animateValue方法啊，我们不是ObjectAnimator吗？哦靠，对啊！

```
//ObjectAnimator
@CallSuper
@Override
void animateValue(float fraction) {
    ...
    super.animateValue(fraction);
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].setAnimatedValue(target);
    }
}
```
果然，我们经验这么老道的，直接看PropertyValuesHolder的setAnimatedValue方法：

```
//PropertyValuesHolder
void setAnimatedValue(Object target) {
    if (mProperty != null) {
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = getAnimatedValue();
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```

mProperty这个变量还记得吧？OK，我们的数据流向已经很明白了。

```
//PropertyValuesHolder
mProperty.set(target, getAnimatedValue());
//BezierTransition/PropPosition
public void set(View view, PointF value)
```

这里再补充一下，动画是如何结束的。刚才一直沿着线索走因而没有过多解释。看这里：

```
//ValueAnimator
if (finished) {
    endAnimation();
}

private void endAnimation() {
    ...
    removeAnimationCallback();
	...
}

private void removeAnimationCallback() {
    ...
    getAnimationHandler().removeCallback(this);
}

//AnimationHandler
public void removeCallback(AnimationFrameCallback callback) {
    ...
    int id = mAnimationCallbacks.indexOf(callback);
    if (id >= 0) {
        mAnimationCallbacks.set(id, null);
        mListDirty = true;
    }
}

//AnimationHandler/Choreographer.FrameCallback
if (mAnimationCallbacks.size() > 0) {
    getProvider().postFrameCallback(this);
}

```
好像哪里不对，好像只有当mAnimationCallbacks长度等于0的时候才停止发信号，但我们这里是置null，要知道ArrayList是允许添加null的。我们是不是哪里漏分析了？我们向前仔细检查一下：

```
//AnimationHandler
private void doAnimationFrame(long frameTime) {
    ...
    for (int i = 0; i < size; i++) {
        final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
        ...
        callback.doAnimationFrame(frameTime);
        ...
    }
	cleanUpList();
}
```

cleanUpList方法，我们貌似没有分析过：

```
//AnimationHandler
private void cleanUpList() {
    if (mListDirty) {
        for (int i = mAnimationCallbacks.size() - 1; i >= 0; i--) {
            if (mAnimationCallbacks.get(i) == null) {
                mAnimationCallbacks.remove(i);
            }
        }
        mListDirty = false;
    }
}
```

到这里，一个动画的循环算是结束了，当然我们还漏了native的没有分析，再回顾一下，调动了哪个native方法，

```
//DisplayEventReceiver
private static native void nativeScheduleVsync(long receiverPtr);
```

由于篇幅原因，而且并不是java层的东西，所以省略了native的分析，如果感兴趣，可以联系我，如果人多，我再把它填进去。调用native的nativeScheduleVsync方法后，最终会回调DisplayEventReceiver的dispatchVsync方法：

```
//DisplayEventReceiver
private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
    onVsync(timestampNanos, builtInDisplayId, frame);
}
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
}
```
发现是空实现，那必然子类实现了其方法，

```
//Choreographer/FrameDisplayEventReceiver
@Override
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
	...
    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}
//Choreographer/FrameHandler
case MSG_DO_FRAME:
    doFrame(System.nanoTime(), 0);
    break;
```
到doFrame方法之后都是我们熟悉的流程了，有同学问Message都没指定what，为啥是这个，因为：

```
//Choreographer
private static final int MSG_DO_FRAME = 0;
```

### 流程图
关于整个属性动画执行过程基本上算是分析完毕，这里再用流程图做个总结。

![](/img/object_animator_chart.png)

### 骚聊
每次到这里都应该是很开心的，但源码分析确实太枯燥了，看得想睡觉。那是不是理解了本文算是理解属性动画源码了呢？那远远不够，这只不过让你知道个动画执行过程，核心类是Choreographer，而关于动画的源码分析可以更深入，看好你哦！

要记住动画的核心并不是技术有多么牛皮，而是创意！

最后，感谢一直支持我的人！
### 传送门
Github：[https://github.com/crazysunj/](https://github.com/crazysunj/)

