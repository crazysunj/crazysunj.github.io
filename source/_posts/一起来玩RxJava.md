---
title: 一起来玩RxJava
toc: true
date: 2019-05-04 21:51:45
tags: [RxJava]
---
## 背景
项目中没有用到和RxJava配合的库；使用RxJava的库是喜欢这种快速切换线程以及声明式的编程方式；但RxJava的功能以及操作符过于庞大，难于全面掌握而导致滥用，本人表示好多操作符也是一脸懵逼，主要归结于平时常用的几个操作符已经满足我们的开发；那么问题来了，RxJava是什么？再见；了解RxJava，喜欢这种简洁，优雅的代码风格，又不想舍弃怎么办？今天我就跟大家一起来打造一个适合自己的异步响应式库，从中我们也可以更加了解RxJava的一些思想。
## SimpleFlowable
今天我们打造的库姑且命名为SimpleFlowable，但却并不具有处理背压的功能（RxJava2从原来Observable抽离出来处理背压的命名其为Flowable），略微尴尬。这里简单介绍一下背压的概念，也是RxJava1暴露出来的一个缺点。
	
> 被观察者发送事件的速度大于观察者接收事件的速度，其中观察者会创建一个无限制大小的缓冲池存储未接收的事件，因此当存储的事件越来越多时就会导致OOM。

简单举例：

```
Observable
        .create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                int i = 0;
                while (true) {
                    // 无限发射数据
                    e.onNext(i++);
                }
            }
        })
        .subscribeOn(Schedulers.newThread())
        .observeOn(Schedulers.newThread())
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                // 每隔5s处理事件，期间将有大量的事件被存储
                Thread.sleep(5000);
            }
        });
```
<!--  more-->
### 基础框架
本文基于版本2.2.8分析。

RxJava是基于观察者模式，这里简单介绍一下：
> 观察者订阅，即在被观察者注册，被观察者发射数据，通知注册的观察者，观察者接收处理事件，Android中最常见的应用是OnClickListener。

现在我们看一下最原始的用法是怎样的。

```
Observable
        .create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                // 发射事件
            }
        })
        // 观察者订阅事件
        .subscribe(new Observer<Integer>() 				
            ...
            @Override
            public void onNext(Integer integer) {
                // 接收事件
            }
            ...
        });
```

这里有点不符合模式的是RxJava采用的被观察者去订阅观察者，但也不难解释，完全为了支持责任链模式。

那么，这样的写法，流到底是怎么走的？让我们一起走进源码（简化版）。

```
// Observable
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    // 创建一个被观察者对象，传入事件发射处理接口
    return new ObservableCreate<T>(source);
}

// ObservableCreate
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
    ...
}

// Observable
public abstract class Observable<T> implements ObservableSource<T>

// ObservableSource
public interface ObservableSource<T> {
    // 被观察者订阅观察者（实际应该反过来，前面已经解释）
    void subscribe(@NonNull Observer<? super T> observer);
}
```

从类结构分析，创建的ObservableCreate返回的父类指向是Observable，说明在Observable中已经实现了subscribe。

```
// Observable#subscribe
public final void subscribe(Observer<? super T> observer) {
    ...
    subscribeActual(observer);
    ...
}
// Observable#subscribeActual
// 抽象方法，肯定是在ObservableCreate中实现
protected abstract void subscribeActual(Observer<? super T> observer);

// ObservableCreate#subscribeActual
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);
    ...
    // 在这里，发射事件接口与观察者关联，而观察者的包装类就是发射事件接口的参数回调，在原始用法中可见
    source.subscribe(parent);
    ...
}
```

在这里介绍一下我们的观察者Observer，其结构是这样的：

```
public interface Observer<T> {
    // 这里主要用于取消订阅
    void onSubscribe(@NonNull Disposable d);
    // 这里就是我们接收并处理事件的地方
    void onNext(@NonNull T t);
    // 接收事件过程中抛出异常
    void onError(@NonNull Throwable e);
    // 整一个流事件完成
    void onComplete();
}
```

再来看看Observer的包装类CreateEmitter：

```
// ObservableCreate.CreateEmitter
// 其实看完这就差不多了解了整个流事件的一般走向，所以我们只要在create的发射事件回调去调用onNext就可以发射事件（不限次数），发射完最好再调用一次onComplete表明发射结束
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
    ...
    // 传入的观察者
    final Observer<? super T> observer;
    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }
    
    @Override
    public void onNext(T t) {
        ...
        observer.onNext(t);
        ...
    }
    
    @Override
    public void onError(Throwable t) {
        ...
        observer.onError(t);
        ...
    }
    
    @Override
    public void onComplete() {
        ...
        observer.onComplete();
        ...
    }
    ...
    // 用于取消订阅
    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }
	
    // 用于判断是否已经取消，因此我们在create发射事件的回调中最好判断以下当前流是否被取消，避免不必要的工作
    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }
    ...   	
}

// 以下是CreateEmitter继承结构
public interface ObservableEmitter<T> extends Emitter<T>

public interface Emitter<T> {
    void onNext(@NonNull T value);
    void onError(@NonNull Throwable error);
    void onComplete();
}

public interface Disposable {
    void dispose();
    boolean isDisposed();
}

// DisposableHelper，枚举类，DISPOSED为成员（可以说被取消后都默认置为此）
public static boolean dispose(AtomicReference<Disposable> field) {
    Disposable current = field.get();
    Disposable d = DISPOSED;
    if (current != d) {
        // 重置为DISPOSED
        current = field.getAndSet(d);
        if (current != d) {
            // 自旋取消
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
    return false;
}
```

这里RxJava对订阅取消处理采用的是AtomicReference，这是一种读和写都是原子性的对象引用变量，但我们并没有看到初始化的地方（构造函数或者set操作），也就是说其默认为null，如果被取消，就会被置为DISPOSED。

这里对取消订阅再补充一点，我们平时还有一种常见的用法是这样的：

```
// 我们可以获取Disposable接口用于取消订阅，不难猜测Observer实现了Disposable，然后再return。
Disposable disposable = Observable
        .create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                // 发射事件
            }
        })
        // 当然还有第4个参数，其用法跟直接传入一个Observer一样
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                // 接收事件
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {

            }
        }, new Action() {
            @Override
            public void run() throws Exception {

            }
        });
```

到这里，常见的用法我们已经写完了，有点流的感觉了，是不是有点小成就呢？是的，现在的库代码都是以上贴出来的，省略部分是RxJava的细节处理，我们统统不要，因为这个。。。
### 线程调度
这可能是选择RxJava的重要原因之一，优雅的切换线程，我们常常是这么用的：

```
Observable
        .create(...)
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(...);
```

当初RxJava还没普及的时候，你可能只是单纯记住subscribeOn的线程用于create和doOnSubscribe，observeOn的线程用于观察者处理和操作符处理，还给你画了个流的走向，各种实验性日志，各种数字标记，然后你都牢牢记住了，那你曾问过为什么吗？现在我们就一起来揭开它神秘的面纱。

#### subscribeOn
先从subscribeOn开始吧。

```
// Observable
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ...
    return new ObservableSubscribeOn<T>(this, scheduler);
}

// Scheduler
public abstract class Scheduler {
	...
	// 用于创建一个Worker
	@NonNull
    public abstract Worker createWorker();
	...
}

// Scheduler.Worker
// Worker实现了Disposable，不难猜测在我们取消订阅的时候，会调用worker的dispose，用于取消线程的执行
public abstract static class Worker implements Disposable {
	...
	// 执行一个Runnable，底层实现不用管它，无非是一种线程的管理
	@NonNull
    public Disposable schedule(@NonNull Runnable run) {
        return schedule(run, 0L, TimeUnit.NANOSECONDS);
    }
    @NonNull
    public abstract Disposable schedule(@NonNull Runnable run, long delay, @NonNull TimeUnit unit);
    ...
}

// ObservableSubscribeOn
// 不难猜出来ObservableSubscribeOn肯定继承（多级）Observable
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
	...
	@Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
        observer.onSubscribe(parent);
        ...
        // 此处的代码，源代码并不是这样的，层级有点多，大致是这样的，就不都提出来讲解了
        final Worker w = createWorker();
        Runnable task = new Runnable {
        	// source为传入的Observable，如果按照实例代码不就是我们create里面创建的ObservableCreate吗？其执行的subscribe方法线程被切换了
        	// 不难解释以前为什么会得出这样的结论了，按照这样的分析，其实只要是调subscribeOn方法的Observable的线程都会被切换，所以它影响的都是此方法前面的Observable的subscribe方法所处线程
        	// 为了验证这个结论，我们再去看看doOnSubscribe
        	source.subscribe(parent);
        };
        w.schedule(task, ...);
        ...
    }
	...
}

// Observable
// 精简过的
public final Observable<T> doOnSubscribe(Consumer<? super Disposable> onSubscribe) {
    return new ObservableDoOnLifecycle<T>(this, onSubscribe,...);
}

// ObservableDoOnLifecycle
public final class ObservableDoOnLifecycle<T> extends AbstractObservableWithUpstream<T, T> {
    ...
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        // source即我们传入的，呵呵，还是ObservableCreate，onSubscribe的执行在DisposableLambdaObserver的onSubscribe方法中执行，onSubscribe的执行会在ObservableCreate的subscribeActual中执行，忘了的拉倒最上面
        source.subscribe(new DisposableLambdaObserver<T>(observer, onSubscribe, onDispose));
    }
}
```

那么subscribeOn的线程调度影响范围就很简单了，就是前一个Observable的subscribe方法所在的线程，如果一直未改变Observable的源，那么最终会到ObservableCreate中，也就是我们最初的create的Observable对象。

可以改变源吗？Of course，到操作符小节再介绍。

#### observeOn
直接上代码：

```
// Observable
public final Observable<T> observeOn(Scheduler scheduler) {
    return new ObservableObserveOn<T>(this, scheduler, ...);
}

// ObservableObserveOn
// 同样的ObservableObserveOn也会继承Observable（多级）
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
	...
	@Override
    protected void subscribeActual(Observer<? super T> observer) {
        ...
        Scheduler.Worker w = scheduler.createWorker();
        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        ...
    }
	...
}

// ObserveOnObserver
// 着重介绍对象，继承关系是ObserveOnObserver->BasicIntQueueDisposable->AtomicInteger
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
    ...
    // 线程执行单元
    final Scheduler.Worker worker;
    // 存储发射事件数据队列
    SimpleQueue<T> queue;
    // 记录error
    Throwable error;
    // 标记是否完成
    volatile boolean done;
    // 标记是否取消
    volatile boolean disposed;
    ...
    @Override
    public void onNext(T t) {
        if (done) {
            // 完成状态将不再执行
            return;
        }
        ...
        // 加入队列
        queue.offer(t);
        ...			
        schedule();
    }

    @Override
    public void onError(Throwable t) {
        if (done) {
            // 完成状态将不再执行
            return;
        }
        // 记录error
        error = t;
        // 标记完成
        done = true;
        schedule();
    }

    @Override
    public void onComplete() {
        if (done) {
            // 完成状态将不再执行
            return;
        }
        // 标记完成
        done = true;
        schedule();
    }
    ...
    void schedule() {
        // 如果当前值是0就执行任务并作加1处理
        if (getAndIncrement() == 0) {
            // 如果在create中作延时操作，那么在延时前的任务可能全部被加入队列，这里可能只执行一次
            // 如果这里可能被多次执行，那么线程可能会是多个，如果要保证都是同一个线程，那么在worker里面必须特殊处理
            // 这里简单介绍RxJava是怎么处理的
            // RxJava依然用的是Java的线程池ScheduledThreadPoolExecutor，其核心线程数量为1，结合DelayedWorkQueue可以保证是同一个线程，玩得很6
            worker.schedule(this);
        }
    }
    ...
    @Override
    public void run() {
        ...
        drainNormal();
        ...
    }
    ...
    void drainNormal() {
        // 用于记录队列数据发射情况，值为0时表示数据暂时发射完
        // 暂时是因为在发射的时候可以作延时操作，那么线程将被中断，但其实数据并没有被发射完，即并没有结束
        int missed = 1;
        // 队列赋值
        final SimpleQueue<T> q = queue;
        // 观察者赋值
        final Observer<? super T> a = downstream;

        for (;;) {
            // 第一层无限循环
            // 检查是否结束
            if (checkTerminated(done, q.isEmpty(), a)) {
                return;
            }
            // 第二层无限循环
            for (;;) {
                boolean d = done;
                T v;

                try {
                    // 队列取出数据
                    v = q.poll();
                } catch (Throwable ex) {
                    // 如果抛出异常，那么以异常流程执行
                    Exceptions.throwIfFatal(ex);
                    disposed = true;
                    upstream.dispose();
                    q.clear();
                    a.onError(ex);
                    worker.dispose();
                    return;
                }
                boolean empty = v == null;
                // 这里empty除了队列为空之外还可能是数据为null，从checkTerminated方法可知empty的时候并不会清空队列，那么该队列不允许null存在
                // 在被观察者的发射事件接口，注意尽量不要发射null
                if (checkTerminated(d, empty, a)) {
                    return;
                }
                // 如果数据为null直接中断第二层循环，否则观察者回调事件
                if (empty) {
                    break;
                }
                a.onNext(v);
            }
            // 循环结束将减去missed，默认是1，如果队列为长度为1减1就为0直接宣告结束
            // 如果队列不为1，可能再执行一遍第二层循环直至此时队列任务全部被执行宣告循环结束
            missed = addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }
    ...
    // 顾名思义，检查是否结束
    boolean checkTerminated(boolean d, boolean empty, Observer<? super T> a) {
        if (disposed) {
            // 如果订阅被取消，那么清空队列并标识结束
            queue.clear();
            return true;
        }
        if (d) {
            // error赋值
            Throwable e = error;
            ...
            if (e != null) {
                // 如果error不为空，那么标识订阅已被取消，队列清空，观察者回调onError，线程执行器结束，标识结束
                disposed = true;
                queue.clear();
                a.onError(e);
                worker.dispose();
                return true;
            } else
            if (empty) {
                // 如果empty为true，那么标识订阅已被取消，观察者回调onComplete()，线程执行器结束，标识结束
                disposed = true;
                a.onComplete();
                worker.dispose();
                return true;
            }
            ...
        }
        // 如果没有完成，均标识为未结束
        return false;
    }
    ...
}
```

按照我们前面分析的，一个Observer将会被包裹在另一个Observer中，所以observeOn影响的是其后的Observer接收事件的线程。

### 操作符
这里我们举最常见的map和flatMap，据我了解，大多开发也只用这两个，你说尬不尬。

#### map
上代码，很easy：

```
// Observable
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    return ObservableMap<T, R>(this, mapper);
}

public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
	...
	@Override
    public void subscribeActual(Observer<? super U> t) {
        // function即传入的mapper，map操作符依然用的传入的Observable，并没有改变
        source.subscribe(new MapObserver<T, U>(t, function));
    }
	...
	static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
		...
		@Override
        public void onNext(T t) {
            ...
            // 相当好理解，只做一下convert，然后再发射数据
            U v = mapper.apply(t);
            downstream.onNext(v);
            ...
        }
		...
	}
}
```

#### flatMap
上，上，上：

```
// Observable
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper) {
    return new ObservableFlatMap<T, R>(this, ...);
}
// ObservableFlatMap
public final class ObservableFlatMap<T, U> extends AbstractObservableWithUpstream<T, U> {
	...
	@Override
    public void subscribeActual(Observer<? super U> t) {
        ...
        // 这里不用管maxConcurrency，因为我们是默认的即int最大值
        // 这里我们发现flatMap任然用的是source，所以线程调度并不能影响flatMap里面创建的Observable
        source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }
	...
	static final class MergeObserver<T, U> extends AtomicInteger implements Disposable, Observer<T> {
		...
		@Override
        public void onNext(T t) {
            ...
            // 这里面的处理是很复杂的，大致原理就这样，这里同样要处理发射的数据很多的情况，在InnerObserver中
            // 原因与observeOn类似，不可能每次执行一次onNext都需要重新创建一个Observable，而是通过队列取出发射
            ObservableSource<? extends U> p;
            p = mapper.apply(t);
            InnerObserver<T, U> inner = new InnerObserver<T, U>(this, ...);
            p.subscribe(inner);
            ...
        }
		...
	}
	...
}
```

#### compose
那么有没有一个操作符可以修改源的呢？有，就是compose。原理也很简单：

```
// Observable
// 精简版，ObservableFromUnsafeSource同样继承Observable，这样整个流就替换成ObservableTransformer接口返回的Observable
// 注意是替换整个流，假设你compose里面不再使用原来的流，那么compose之前的将毫无意义，已经分析这么多了，就不用我说了吧
public final <R> Observable<R> compose(ObservableTransformer<? super T, ? extends R> composer) {
    return new ObservableFromUnsafeSource<T>(composer.apply(this));
}
```

## 感言
整篇下来，我们简陋的RxJava库SimpleFlowable算是完成。整篇文章体现了RxJava的核心思想(异步响应式)，但非难点，更复杂的是实现的各种细节，这里面的精妙之处还需我们细细体会，对我们的编程思想是很有帮助的。

下期再会！