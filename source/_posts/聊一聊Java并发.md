---
title: 聊一聊Java并发
toc: true
date: 2018-08-03 17:50:22
tags: [并发,synchronized,死锁,volatile,final,JMM]
---
## 前言
Java并发无处不在，如果你是开发Android的。因为Android的机制必然会导致多线程的存在，那么应用程序也必然是并发的。那么单线程不支持并发是吗？并不然，至少Java是支持的，有兴趣的可以了解下协程，这里不多解释了。
<!--more-->
## 线程
你以为我会解释什么是线程？不可能！这里提一下线程的6种状态：NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED。

这里再给出状态之间切换的几种情况，大家照着图摸索一下就差不多了。

![](/img/thread_status.png)

这里主要就是线程之间的通信。
## 并发
说起并发的概念，我的理解是有变化的，由于自己并不是研究并发这一块，所以很多知识也是靠阅读而来，这里仅供参考。

一开始，我认为并发就是一个点在同一时刻被多个点访问。同时也有一个新的概念就是并行，这个比较好理解，单纯就是物理上的同时执行，应该没什么争论。后来，我阅读到一篇文章，文章链接忘了，没收藏，上面提到大佬Rob Pike的一个观点，他认为：
> Concurrency is about dealing with lots of things at once.
> Parallelism is about doing lots of things at once.

翻译有难度，自己必须有很深的理解，故还是给英文。

用我自己的话来讲就是，并发是一种程序上的设计，支持处理多个任务，如果是这样，那么后者并行其实就是并发的一种状态。我更相信这个观点，我们并发编程不就是说我们的应用程序支持多线程吗？可能我自己的话不太严谨，如果我这样说，并发就是在逻辑上能够让多个任务交替执行的程序设计。

这样说吧，只要你的程序支持多线程，那么你就是在并发编程，因为绝大多数的并发都是多线程。而并发编程一个非常大的难点就线程是安全。
## 线程安全
以下定义摘自维基百科：
> 线程安全是编程中的术语，指某个函数、函数库在多线程环境中被调用时，能够正确地处理多个线程之间的共享变量，使程序功能正确完成。

简单点说，执行一段代码，运行结果与预知结果一致，那么这段代码就是线程安全的，但其实不太严谨，可以试着这么理解。

以下几个小节，可能需要你掌握Java内存的概念，如果你不太清楚，可以看我这篇文章[揭开Java内存管理的面纱](http://crazysunj.com/2018/01/28/%E6%8F%AD%E5%BC%80Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%9A%84%E9%9D%A2%E7%BA%B1/)。

在此之前，我们先来了解一个常谈的点：死锁。
### 死锁
可能口头讲，有点吃力，我们来看一段代码（摘自网上，懒得打了，都差不多的）：

```
public class DeadLock {
    public static String obj1 = "obj1";
    public static String obj2 = "obj2";
    public static void main(String[] args){
        Thread a = new Thread(new Lock1());
        Thread b = new Thread(new Lock2());
        a.start();
        b.start();
    }    
}
class Lock1 implements Runnable{
    @Override
    public void run(){
        try{
            System.out.println("Lock1 running");
            while(true){
                synchronized(DeadLock.obj1){
                    System.out.println("Lock1 lock obj1");
                    Thread.sleep(3000);//获取obj1后先等一会儿，让Lock2有足够的时间锁住obj2
                    synchronized(DeadLock.obj2){
                        System.out.println("Lock1 lock obj2");
                    }
                }
            }
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
class Lock2 implements Runnable{
    @Override
    public void run(){
        try{
            System.out.println("Lock2 running");
            while(true){
                synchronized(DeadLock.obj2){
                    System.out.println("Lock2 lock obj2");
                    Thread.sleep(3000);
                    synchronized(DeadLock.obj1){
                        System.out.println("Lock2 lock obj1");
                    }
                }
            }
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
```
执行结果：

```
Lock1 running
Lock2 running
Lock1 lock obj1
Lock2 lock obj2
```
这就是死锁，直接导致两个线程都没法继续执行，处于等待对方释放锁的状态（Lock1获取锁DeadLock.obj1，Lock2获取锁DeadLock.obj2，此时Lock1又想获取锁DeadLock.obj2，但是已经被Lock2获取，锁只有一把，只能等到它不用了才能获取，Lock2获取锁DeadLock.obj1同理）。

避免死锁也很简单：避免同一把锁被多个线程获取；避免一个线程获取多把锁；可以使用Lock替代synchronized，因为Lock有超时机制，超时自动释放锁。

讲死锁，其实就是想引出Java的关键字synchronized，属于内置锁。synchronized可以实现代码的同步达到线程安全。
### synchronized
#### 原理
synchronized可以用来修饰方法或者块。像上面死锁的例子就是修饰块，我们可以看到锁是我们自定义的，而修饰方法的时候，synchronized被当成修饰符，那么锁是谁呢？静态方法的锁是当前类的Class对象，普通方法的锁是当前的实例对象。

那么JVM是如何知道的呢？因为JVM看的不是我们的Java代码，而是字节码。如果是同步代码块，那么块中的逻辑将被monitorenter和monitorexit字节码指令包裹，从命名也很容易知道，前者表示同步的开始，后者表示同步的结束。如果是同步方法，那么标识就跟public一样，JVM识别public方法，是靠访问标志ACC\_PUBLIC，同理，synchronized是靠访问标志ACC\_SYNCHRONIZED。
#### 锁
synchronized实现同步，需要一把锁，为什么对象可以作为锁，JVM是怎么识别的呢？这个我们得了解一个基础知识，对象头，它属于对象的数据结构之一，又称markword。对象头在32位或者64位JVM中长度是不一样的，这里我们不考虑这些，简化列出下面表格：

| 锁状态 | 锁标志位/是否为偏向锁 | 存储数据 |
| :------:| :------: | :------: |
| 无锁 | 01/0 | hashCode、分代年龄 |
| 偏向锁 | 01/1 | 线程id、时间戳、分代年龄 |
| 轻量级锁 | 00/- | 指向栈中锁记录的指针 |
| 重量级锁 | 10/- | 指向互斥量（重量级锁）的指针 |

可能你对上面的一些名词感到懵逼，正常！我慢慢解释。
jdk1.6引入了偏向锁和轻量级锁，减少了锁的性能消耗，这也是Lock为啥出来的原因，synchronized现在还不够强大。其实不难猜出来，它们其实有level的概念，level越高所需性能消耗越高，但功能肯定越强大。其level从小到大为偏向锁、轻量级锁、重量级锁，且它们有一套升级规则。
##### 偏向锁
偏向锁顾名思义就是偏向，偏向某一线程。大多数情况，我们只有一个线程在访问，那么没必要去获取锁，即使获取释放后，下次还是这个线程，那么又要获取一次锁，如果多次呢？所以，不存在竞争的时候，那么将会是偏向锁，且不存在同步。那么，偏向锁是如何获取？又是如何升级的呢？下面流程图给你答案：

![](/img/biased_lock.png)

##### 轻量级锁
轻量级锁真的是很轻量的，它可以达到同步的效果，但属于轻量的，通过CAS操作和自旋在一定程度达到同步。CAS操作在Java并发中无处不在，下文会介绍。轻量级锁的具体流程如下：

![](/img/lightweight_lock.png)

##### 重量级锁
重要级锁之所以重量，是因为它需要在用户态和内核态来回切换，成本较高。

##### 总结
| 锁 | 优点 | 缺点 | 适用场景 |
| :------:| :------: | :------: | :------: |
| 偏向锁 | 加锁和解锁不需要额外的消耗，和执行非同步方法相比仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问同步块场景 |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度 | 如果始终得不到锁竞争的线程，使用自旋会消耗CPU | 追求响应时间，同步块执行速度非常快 |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU | 线程阻塞，响应时间缓慢 | 追求吞吐量，同步块执行速度较长 |


synchronized是我们最了解的，所以放在了最前面，synchronized之所以可以达到线程安全，凭我们的经验是因为一旦有线程进入同步块，那么其它线程必须等到锁释放才能进行访问，那么可以理解成同步块就是一个原子，不可再分割，要么不执行，一旦执行，全部执行。
### 三大特性
#### 原子性
如果一个操作不可分割，那么我们就可以认为这是一个原子操作。除了synchronized以外，Java还有其它常见的做法来达到原子操作。在此之前，我们更深理解何谓原子操作，看下面代码：

```
1. a = 1;
2. a++;
```
两句代码都很简洁，但第一句是原子操作，因为它只有赋值的操作；第二句中，其实干了这么多事，先读取a的值，然后进行a+1计算，最后把计算结果赋值给a。

这里有一个疑问我一直没讲，为什么不保证原子性，线程就是不安全的呢？这里你得了解Java内存模型(JMM)。这里给个简单模型图助你理解。

![](/img/jmm_model.png)

我们线程操作的其实是变量的副本，那么会导致什么情况发生呢？这里我们举个简单的例子：线程A执行a++，同时线程B也执行a++，比如执行前a的值为1，那么根据我们刚才分析，读取a的值得时候，可能线程A和B读取到的值都是1，那么在工作内存中计算后都是2，而我们的预期的结果应该是3。可能线程B在执行计算的时候，线程A已经得出结果是2了，此时a的值是2，但是线程B是不知道的，计算的时候还是用当时读取的值1来进行计算。不知道这种文字解释能否明白。

经过上面解释，这样可以有个轻量级的原子操作，我们只要预期的值跟实际内存地址的值比较，如果一样，那么就可以更新值，这就是CAS操作，大致函数可以表示为：
> CAS(V,E,N) // V表示内存地址的值 E表示预期的值 N表示要更新的值

如果V和E相等，那么就把V修改为N，这样说可能有点抽象，我们来看一下Java提供的原子类AtomicInteger源码中的处理：

```
// AtomicInteger
private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
private static final long VALUE; // 内存中的值
static {
    try {
        VALUE = U.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (ReflectiveOperationException e) {
        throw new Error(e);
    }
}
private volatile int value; // 当前值
public final int getAndAdd(int delta) {
    return U.getAndAddInt(this, VALUE, delta);
}

// sun.misc.Unsafe
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```
getAndAdd方法显然是我们的加法，比如加2，那么参数就是2。底层调用的是Unsafe类的compareAndSwapInt方法去实现CAS操作，var1就是我们AtomicInteger的实例，var2是内存地址的值，var5是预期值（其实就是当前Java内存中的值），最后一个参数就是我们要更新的值了。那么问题又来，如果说线程操作的数据是有缓存的话，那么这种方法肯定不成立啊，是的，确实是这样，可能无限循环下去了。但有没有注意到我们当前的值value有个修饰符volatile。它可以保证无论在哪个线程看到的值永远是最新的。

我们先说说CAS的缺陷吧，很明显，如果一直CAS不成功，那么开销也不小；还有一个致命的缺点就是只能操作一个变量（当然你可以合成一个值操作）；最后一个缺点就是著名的ABA问题，试想，如果变量a的值是A，然后a的值更新为B，最后再更新回A，但是你比较的时候，认为a的没有变动过的，因为值是一样的，常见的做法是加个版本号。
#### 可见性
我们的修饰符volatile可以保证所有线程操作的值都是最新的，并不是副本。其实这就是可见性，当一个变量被操作后，其它线程可以立即知道最新的值。那么问题来了，既然，假设所有的变量都用volatile修饰，难道不能保证原子性吗？如果真是这样，AtomicInteger根本不用CAS操作，多此一举。我们举个简单的例子来说明一下，为什么不能保证原子性。例子跟我们聊JMM和原子性的例子一样，即使volatile可以保证可见性，但是例如像i++这种操作其实是多步的，首先它得读取i的值，这个没问题，线程A和线程B都可以读取到最新的值，假设值i是1，那么线程A和B都能读取到且值是最新的1，但是后续的操作双方都不知道的，因为各自都在各自的工作内存中，即使A完成了+1操作，并写入，主内存也就是2而已，线程B计算完依然是2，除非线程B再读取一遍，读取到2，然后进行+1操作，结果正确。从这个例子，我相信你对CAS的理解更深了。volatile只能保证读或者说写是原子的。

更深一点理解volatile就是对内存屏障的运用（Memory Barrier）。

| 屏障类型 | 指令示例 | 说明 |
| :------:| :------: | :------: | 
| LoadLoad Barriers | Load1; LoadLoad; Load2 | 确保Load1数据的装载先于Load2及所有后续装载指令的装载 |
| StoreStore Barriers | Store1; StoreStore; Store2 | 确保Store1数据对其他处理器可见（刷新到内存）先于Store2及所有后续存储指令的存储 |
| LoadStore Barriers | Load1; LoadStore; Store2 | 确保Load1数据装载先于Store2及所有后续的存储指令刷新到内存 |
| StoreLoad Barriers | Store1; StoreLoad; Load2 | 确保Store1数据对其他处理器变得可见（指刷新到内存），之前于Load2及所有后续装载指令的装载。StoreLoad Barriers会使该屏障之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令 |

一般来说用StoreLoad Barriers就够了，它具备其它三个的效果，但开销最大。

现在我们举个例子来理解内存屏障。

```
public class VolatileBarrierExample {  
    long a;  
    volatile long v1 = 1;  
    volatile long v2 = 1;  

    void readAndWrite(){  
        long j = v1;  
        long i = v2;  
        a = i + j;  
        v1 = i + 1;  
        long v = v1;  
        v2 = j * 2;  
    }     
    ...    
}
```

例如上面代码中readAndWrite方法最终的执行顺序是这样的：

![](/img/memory_barrier_volatile.png)

那么为什么需要内存屏障来限制执行顺序？因为编译器和处理器经常会对指令进行重排序。重排序分成三种类型：

1. 编译器优化的重排序。编译器在不改变单线程程序语义放入前提下，可以重新安排语句的执行顺序。
2. 指令级并行的重排序。现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
3. 内存系统的重排序。由于处理器使用缓存和读写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

那如何得到最终的执行指针序列？

![](/img/reordering_flow.png)

#### 有序性
即程序执行的顺序按照代码的先后顺序执行。

一般来说，除了synchronized和volatile能保证有序性外，final同样可以。前两者我相信你已经很清楚为什么可以，在介绍final之前，先来理解有序性为啥会影响到线程安全。先看如下代码：

假设线程A的代码如下：

```
a = new A();
b = true;
```

线程B的代码如下：

```
if (b) {
  c = a.c;
}
```

正常情况，线程A先执行，然后线程B再执行，一点毛病都没有，如果是线程A中的代码重排序之后是这样的执行顺序：

```
b = true;
a = new A();
```

当线程A执行到b=true的时候阻塞了，然后线程B开始了工作，此时a并没有初始化，所以直接报了空指针，但是莫名其妙。

这就是有序性导致的安全问题。那么为啥final也能保证有序性呢？

```
public class FinalExample {
    int i;                            //普通变量
    final int j;                      //final变量
    static FinalExample obj;

    public void FinalExample () {     //构造函数
        i = 1;                        //写普通域
        j = 2;                        //写final域
    }

    public static void writer () {    //写线程A执行
        obj = new FinalExample ();
    }

    public static void reader () {       //读线程B执行
        FinalExample object = obj;       //读对象引用
        int a = object.i;                //读普通域
        int b = object.j;                //读final域
    }
}
```

比如这样的代码，我们这里假设线程A先执行，当给分配完内存空间后（此时还没有完全初始化），此时线程B也开始执行，那么obj其实不为空，那么a取到的是0，而b取到的是2。应该比较好理解了吧？我们经典的双重检查锁单例写法亦是如此。其实追根到底，还是我们的内存屏障。
## 结束语
Java的并发知识很多，今天聊的挑了几个重点，希望对大家有帮助吧，也算是自己回顾一下，后续可能会对ConcurrentHashMap源码解析，毕竟这是面试常客，且理解ConcurrentHashMap可以对并发更深入，ConcurrentHashMap在设计层面也非常优秀，值得学习。



