---
title: 玩一玩Okio源码
toc: true
date: 2018-07-08 18:42:04
tags: 【Okio,源码】
---

## 前言
前篇[《玩一玩OkHttp缓存源码》](http://crazysunj.com/2018/06/24/%E7%8E%A9%E4%B8%80%E7%8E%A9OkHttp%E7%BC%93%E5%AD%98%E6%BA%90%E7%A0%81/)分析OkHttp缓存源码的时候涉及到缓存的读写，而OkHttp底层采用Okio实现，所以特地写了一篇文章介绍Okio，也算自己的一个总结。我们现在主要知道它是怎么一个工作过程，至于架构之美有点难度，即使你总结出Okio的架构，运用自如，举一反三还是很难。很多优秀框架亦是如此，最重要的还是创意。

## 分析
废话不多说，老规矩，从实际到理论，不太喜欢上来就直接架构分析，所以跟着我的思路一边阅读文章一边阅读源码更佳。可能一开始比较懵逼，但是过了一遍再过一遍时就很清晰了。由于读和写是一个反过程，因此我们这里只以写为例。
<!--  more-->
### 数据写入
如果我们想在一个文件里写一点内容，那么可以这样做：

```
new Thread(() -> {
    File test = new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS), "test.txt");
    try {
        BufferedSink bufferedSink = Okio.buffer(Okio.sink(test));
        bufferedSink.writeUtf8("不知道写啥");
        bufferedSink.close();
        runOnUiThread(() -> {
            Toast.makeText(this, "写入成功", Toast.LENGTH_SHORT).show();
        });
    } catch (IOException e) {
        e.printStackTrace();
        runOnUiThread(() -> {
            Toast.makeText(this, "写入失败", Toast.LENGTH_SHORT).show();
        });
    }
}).start();
```

OK，我们一步步分析，首先我们得知道要写入的文件test，这个比较好理解。然后把文件test传入了Okio的静态方法sink。不太懂？那就看源码，源码是最好的老师。

```
//Okio
//文件写入，写入内容会覆盖(java io的基础知识)
public static Sink sink(File file) throws FileNotFoundException {
    ...
    return sink(new FileOutputStream(file));
}

//文件写入，写入内容追加
public static Sink appendingSink(File file) throws FileNotFoundException {
    ...
    return sink(new FileOutputStream(file, true));
}

//最终都会调用
public static Sink sink(OutputStream out) {
    return sink(out, new Timeout());
}

//特例Socket
public static Sink sink(Socket socket) throws IOException {
    ...
    AsyncTimeout timeout = timeout(socket);
    Sink sink = sink(socket.getOutputStream(), timeout);
    return timeout.sink(sink);
}
```

即使特例Socket我们看代码也会调用sink(final OutputStream out, final Timeout timeout)。只不过timeout再包装一次，Socket我们先不讲，放在超时机制一起讲。

那么最终的问题都指向两个点：

* Sink是什么玩意
* sink(final OutputStream out, final Timeout timeout)干了什么事

我们一个一个来解决，从我们出发点是写入不难猜出Sink是关于输出的。我们具体看看是怎么回事：

```
/**
 * 接受字节流，可以在网络、磁盘或者说内存中写入。
 */
public interface Sink extends Closeable, Flushable {
  /** 写入字节 */
  void write(Buffer source, long byteCount) throws IOException;

  /** 刷新缓冲 */
  @Override void flush() throws IOException;

  /** 超时 */
  Timeout timeout();

  /** 关闭Sink */
  @Override void close() throws IOException;
}
```
我们现在大致知道Sink是一个写入的接口。同时引入一个新的概念Buffer。暂且记下。

我们再来看看第二个问题：

```
//Okio
private static Sink sink(final OutputStream out, final Timeout timeout) {
    ...
    return new Sink() {
      @Override public void write(Buffer source, long byteCount) throws IOException {
        ...
        while (byteCount > 0) {
          timeout.throwIfReached();
          Segment head = source.head;
          int toCopy = (int) Math.min(byteCount, head.limit - head.pos);
          out.write(head.data, head.pos, toCopy);

          head.pos += toCopy;
          byteCount -= toCopy;
          source.size -= toCopy;

          if (head.pos == head.limit) {
            source.head = head.pop();
            SegmentPool.recycle(head);
          }
        }
      }

      @Override public void flush() throws IOException {
        out.flush();
      }

      @Override public void close() throws IOException {
        out.close();
      }

      @Override public Timeout timeout() {
        return timeout;
      }

      @Override public String toString() {
        return "sink(" + out + ")";
      }
    };
}
```

除了write方法相对好理解，而write方法中Buffer卡主了我们，从头到尾都在操作Buffer。看来不理解是不行了，简单推理一下，write是我们返回Sink对象的一个方法，那么拥有这个Sink对象的类肯定会调用write方法。我们再回到用例。

```
BufferedSink bufferedSink = Okio.buffer(Okio.sink(test));
```
刚刚我们简单过了一下Okio的sink方法，显然返回的Sink对象以Okio的buffer方法参数身份传入。

```
//Okio
public static BufferedSink buffer(Sink sink) {
    return new RealBufferedSink(sink);
}
```

首先分析下返回的是BufferedSink，了解下整体结构。

```
public interface BufferedSink extends Sink, WritableByteChannel {
  /** 返回内部的缓冲 */
  Buffer buffer();
  
  //一堆的write方法，基本分为int、long、String、Source、byte和ByteString(主要负责byte和String的一些转换)
  ....

  /** 将所有的缓冲数据写入底层Sink且强制刷新缓冲 */
  @Override void flush() throws IOException;

  /** 将所有的缓冲数据写入底层Sink，弱于flush方法 */
  BufferedSink emit() throws IOException;

  /** 将完成的Segment写入底层Sink */
  BufferedSink emitCompleteSegments() throws IOException;

  /** 返回一个输出流 */
  OutputStream outputStream();
}
```
表示看注释也是懵逼的，我们回到buffer方法，然后看具体实现类。

```
final class RealBufferedSink implements BufferedSink {
  //缓冲
  public final Buffer buffer = new Buffer();
  //底层Sink
  public final Sink sink;
  当前BufferedSink是否关闭
  boolean closed;

  RealBufferedSink(Sink sink) {
    ...
    this.sink = sink;
  }
  ...
}
```

我们看到我们想要的Buffer，进去参观一下。

```
public final class Buffer implements BufferedSource, BufferedSink, Cloneable, ByteChannel {
  ...
  /** Segment链表头，暂且记下，稍后分析 */
  @Nullable Segment head;
  /** Buffer大小 */
  long size;

  public Buffer() {
  }
  ...
}
```
Buffer实现了BufferedSource和BufferedSink，从我们分析来看BufferedSink的api拥有一堆的write方法，那么猜测是负责写入的类，反之BufferedSource就是负责读取的类，同时实现。那么我们这里大胆猜测Buffer这个类是真正的幕后操手且同时支持读写操作，这样设计应该是为了避免逻辑的重复吧，毕竟读写只是一个反过程。

既然实现这个两接口，那下面必然是一堆实现方法，我们先不看。到目前为止我们已知信息还是很少，因为我们只分析了类的创建，那么只能了解类的结构和一些注释信息，最终还是要回到例子，分析具体的写操作。

```
bufferedSink.writeUtf8("不知道写啥");
```
从我们上面的分析可知，BufferedSink的实现类是RealBufferedSink，那么我们直接看RealBufferedSink的writeUtf8方法。

```
// RealBufferedSink
@Override public BufferedSink writeUtf8(String string, int beginIndex, int endIndex)
      throws IOException {
    ...
    buffer.writeUtf8(string, beginIndex, endIndex);
    return emitCompleteSegments();
}
```
果不其然，buffer是幕后操手。OK，那么我们去看看buffer的writeUtf8方法。

```
// Buffer
@Override public Buffer writeUtf8(String string) {
    return writeUtf8(string, 0, string.length());
}

@Override public Buffer writeUtf8(String string, int beginIndex, int endIndex) {
   ...
   writeByte(...) //好像有点长，我们还是分析里面的writeByte吧
   ...
   return this;
}

@Override public Buffer writeByte(int b) {
    Segment tail = writableSegment(1);
    tail.data[tail.limit++] = (byte) b;
    size += 1;
    return this;
}
```
靠，真的是一环接一环啊，貌似不知道Segment是没法玩了，还记得我们在Buffer成员变量记下的Segment吗？不管怎么样，我们必须知道这玩意是干什么的。

```
//buffer的一段？
final class Segment {
  /** 整个Segment的最大字节数 */
  static final int SIZE = 8192;

  /** 共享数据的最小字节数 */
  static final int SHARE_MINIMUM = 1024;

  /** Segment存储的字节数据 */
  final byte[] data;

  /** Segment可读的起始位置 */
  int pos;

  /** Segment可写的起始位置 */
  int limit;

  /** 数据是否共享 */
  boolean shared;

  /** 是否允许写入 */
  boolean owner;

  /** 下个节点Segment */
  Segment next;

  /** 上个节点Segment */
  Segment prev;

  Segment() {
    this.data = new byte[SIZE];
    this.owner = true;
    this.shared = false;
  }

  Segment(byte[] data, int pos, int limit, boolean shared, boolean owner) {
    this.data = data;
    this.pos = pos;
    this.limit = limit;
    this.shared = shared;
    this.owner = owner;
  }
  ...
}
```

从Segment的数据结构，我们可以了解到Segment的是一个双向链表结构，其次它允许数据共享，最后它可以防止数据的污染而可以设定不允许写入。这是我们现在的已知信息，我们再回到原来的代码处。

```
// Buffer
@Override public Buffer writeByte(int b) {
    Segment tail = writableSegment(1);
    tail.data[tail.limit++] = (byte) b;
    size += 1;
    return this;
}
```

继续追踪writableSegment方法。

```
// Buffer
Segment writableSegment(int minimumCapacity) {
    ...
    if (head == null) {
      head = SegmentPool.take(); 
      return head.next = head.prev = head;
    }

    Segment tail = head.prev;
    if (tail.limit + minimumCapacity > Segment.SIZE || !tail.owner) {
      tail = tail.push(SegmentPool.take()); 
    }
    return tail;
}
```

呵呵，还玩不玩？不过这个看名字就懂啥意思，应该是Segment池，联想一下我们熟悉的线程池。我们进去看看。

```
final class SegmentPool {
  /** 池的最大容量，相当于8个吧？ */
  static final long MAX_SIZE = 64 * 1024; 
  /** 单链表的头 */
  static @Nullable Segment next;
  /** 当前池的大小 */
  static long byteCount;

  private SegmentPool() {
  }

  static Segment take() {
    //若next不为空，取出第一个Segment返回
    synchronized (SegmentPool.class) {
      if (next != null) {
        Segment result = next;
        next = result.next;
        result.next = null;
        byteCount -= Segment.SIZE;
        return result;
      }
    }
    return new Segment(); //如果next为空，那么重新构造一个
  }

  static void recycle(Segment segment) {
    ...
    if (segment.shared) return; //如果是共享的那么不回收
    synchronized (SegmentPool.class) {
      // 池满了也不回收
      if (byteCount + Segment.SIZE > MAX_SIZE) return;
      // 插入到链表头 
      byteCount += Segment.SIZE;
      segment.next = next;
      segment.pos = segment.limit = 0; // 清空数据
      next = segment;
    }
  }
}
```
SegmentPool相对较为简单，我们很快就了解了。OK，我们再回过头来看，writableSegment方法。

```
// Buffer
Segment writableSegment(int minimumCapacity) {
    ...
    if (head == null) {
      // 如果当前Buffer的链表头为空，那么从SegmentPool取一个Segment
      head = SegmentPool.take(); 
      // 形成循环双向链表，可知head是一个循环双向链表
      return head.next = head.prev = head; 
    }

    Segment tail = head.prev; //取出最后一个Segment
    if (tail.limit + minimumCapacity > Segment.SIZE || !tail.owner) {
      // 如果末尾可写位置加将写的字节数已经大于Segment的最大字节数
      // 或者说末尾的Segment不可写，那么都会在末尾的Segment后插入一个Segment
      tail = tail.push(SegmentPool.take()); 
    }
    // 返回可写的Segment
    return tail;
}
```

OK，这里我们可能不太明白的就是Segment是如何插入的，索性把取出代码一起分析了，毕竟是成对的。

```
// Segment
public final Segment push(Segment segment) {
    segment.prev = this; //插入Segment的前一个指向自己
    segment.next = next; //插入后一个指向自己后一个
    next.prev = segment; //自己后一个的前一个指向插入
    next = segment; //自己后一个指向插入，形成双向
    // 返回插入Segment
    return segment;
}

// 不解释了，数据结构基础知识，同上
public final @Nullable Segment pop() {
    Segment result = next != this ? next : null;
    prev.next = next;
    next.prev = prev;
    next = null;
    prev = null;
    // 返回下一个Segment
    return result;
}
```

我们再往上翻，回到写入字节的方法。

```
// Buffer
@Override public Buffer writeByte(int b) {
    Segment tail = writableSegment(1);
    tail.data[tail.limit++] = (byte) b; // 存储数据
    size += 1; // size+1
    return this;
}
```

貌似结束了？好像不是，到这里毕竟只是写入Buffer。而真正拥有流对象的是Sink。我们再往回退，回到RealBufferedSink的writeUtf8方法。

```
// RealBufferedSink
@Override public BufferedSink writeUtf8(String string) throws IOException {
    ...
    buffer.writeUtf8(string); 
    return emitCompleteSegments();
}
```

buffer的writeUtf8方法，我们已经玩的很6了，buffer的主要作用就是把数据存储在自己的head循环双向链表中。我们再来看看emitCompleteSegments方法。

```
// RealBufferedSink
@Override public BufferedSink emitCompleteSegments() throws IOException {
    ...
    long byteCount = buffer.completeSegmentByteCount();
    if (byteCount > 0) sink.write(buffer, byteCount);
    return this;
}

// Buffer
public final long completeSegmentByteCount() {
    long result = size;
    if (result == 0) return 0;
    // 循环链表得到最后一个
    Segment tail = head.prev;
    if (tail.limit < Segment.SIZE && tail.owner) {
      // 如果最后一个Segment可写，且可写位置小于最大容量，那么得减去可写的数据量，估计在最后一个可写的情况不能立即刷新吧
      result -= tail.limit - tail.pos;
    }
    // 返回Buffer已存储的数据大小
    return result;
}
```
那么只剩下最后一句了sink.write(buffer, byteCount)，还记得sink的来历吗？不记得没关系，我们再回顾一遍。

```
BufferedSink bufferedSink = Okio.buffer(Okio.sink(test));

private static Sink sink(final OutputStream out, final Timeout timeout) {
    ...
    return new Sink() {
      ...
    };
}

public static BufferedSink buffer(Sink sink) {
    return new RealBufferedSink(sink);
}

RealBufferedSink(Sink sink) {
    ...
    this.sink = sink;
}
```

可能认真看的同学比较厌恶这样的做法，体谅一下。OK，最终还是回到最初卡主我们的地方。

```
// Okio$Sink
@Override public void write(Buffer source, long byteCount) throws IOException {
    ...
    while (byteCount > 0) {
      timeout.throwIfReached();
      Segment head = source.head; // 取出Buffer的head
      /** 话说(head.limit - head.pos)会大于byteCount吗？可能特殊情况吧 */
      int toCopy = (int) Math.min(byteCount, head.limit - head.pos); 
      out.write(head.data, head.pos, toCopy); // 输出流写入

      head.pos += toCopy; // 可读位置往后偏
      byteCount -= toCopy; // byteCount下调
      source.size -= toCopy; // head的size下调

      if (head.pos == head.limit) {
        // 如果pos后偏后与limit相等，表示该Segment已经用完了
        // 移除head并返回下一个
        source.head = head.pop();
       // 回收head
        SegmentPool.recycle(head);
      }
    }
}
```

到这里差不多了，剩下的flush或者说close底层都是调输出流的对应方法，代码我们也看过了。

我们这里简单总结下我们所分析的okio优点：

* ByteString让我们byte[]与String优雅转换，同时提供了很多字符处理工具
* 数据处理采用链表Segment，扩容简单，SegmentPool支持复用，大大减少了OOM
* Api简洁易用

除此之外，在内存方面做得更优秀，Segment提供压缩和共享功能。其次很重要的一点okio支持超时机制，这一点下一节讲。最后它还提供了很多扩展类，例如GzipSink。我先把Segment的压缩和共享和大家说一说。

这两种功能现在放在写入一个Buffer的时候。具体方法解析：

```
// Segment 压缩
public final void compact() {
    ...
    // 如果不可写，返回
    if (!prev.owner) return; 
    int byteCount = limit - pos; // 当前已用字节数
    int availableByteCount = SIZE - prev.limit + (prev.shared ? 0 : prev.pos); // 前一个可用字节数
    if (byteCount > availableByteCount) return; // 如果前一个装不下，返回
    writeTo(prev, byteCount); // 写入前一个
    pop(); // 移除当前Segment
    SegmentPool.recycle(this); // 回收
}

// 共享
public final Segment split(int byteCount) {
    ...
    Segment prefix;
    // 如果拆分字节大于最小共享数据，那么进行数据共享
    // 否则，利用arraycopy进行数据转移
    if (byteCount >= SHARE_MINIMUM) {
      prefix = sharedCopy();
    } else {
      prefix = SegmentPool.take();
      System.arraycopy(data, pos, prefix.data, 0, byteCount);
    }
    // 处理被拆分的Segment的数据区域
    prefix.limit = prefix.pos + byteCount;
    // 因为被拆分，所以可读往后偏移
    pos += byteCount;
    // 插入到自己前面
    prev.push(prefix);
    return prefix;
}

final Segment sharedCopy() {
    shared = true;
    共享data，且数据不可写入，同时两者皆为共享
    return new Segment(data, pos, limit, true, false);
}
```

算了，算了，再给大家讲解一下一个Segment写入到另一个Segment。

```
public final void writeTo(Segment sink, int byteCount) {
    // 不可写，拒绝
    if (!sink.owner) throw new IllegalArgumentException();
    if (sink.limit + byteCount > SIZE) {
      // 共享，拒绝
      if (sink.shared) throw new IllegalArgumentException();
      // 如果加上写入的字节超过了最大字节数，拒绝
      if (sink.limit + byteCount - sink.pos > SIZE) throw new IllegalArgumentException();
      // 如果可写位置加上写入字节数超过最大字节数，那么牺牲pos往前的数据，并进行数据偏移
      System.arraycopy(sink.data, sink.pos, sink.data, 0, sink.limit - sink.pos);
      sink.limit -= sink.pos;
      sink.pos = 0;
    }
    // 当前数据转移
    System.arraycopy(data, pos, sink.data, sink.limit, byteCount);
    sink.limit += byteCount; //可写位置往后偏
    pos += byteCount; // 数据已经转移，可读位置往后偏
}
```

关于数据写入到这里就结束了，关于写入的核心逻辑我们都已经分析，读取只不过是个反过程，我相信你没有问题。
### 超时机制
超时机制也是Okio的一大亮点，同时它分为同步和异步超时。
#### 同步超时
我们先整理下关于同步超时的线索。

```
// Okio
private static Sink sink(final OutputStream out, final Timeout timeout) {
   ...
   return new Sink() {
      @Override public void write(Buffer source, long byteCount) throws IOException {
        ...
        timeout.throwIfReached();
        ...
      }
      ...
      @Override public Timeout timeout() {
        return timeout;
      }
      ...
    };
}

// timeout是我们默认传进去
public static Sink sink(OutputStream out) {
    return sink(out, new Timeout());
}
```
根据我们已知线索说明BufferedSink会调用sink的timeout方法进行操作。但如果看得仔细的话，会发现BufferedSink接口其实是继承Sink接口的，而Sink接口是有timeout方法，不出意外的话BufferedSink实现这个方法并把sink的timeout传入。

```
// BufferedSink
@Override public Timeout timeout() {
    return sink.timeout();
}
```
简单的推理。然后就没然后了，应该BufferedSink中压根没用，只是单纯实现这个接口。那么，按照这个推理Buffer肯定也实现了这个方法。

```
// Buffer
@Override public Timeout timeout() {
    return Timeout.NONE;
}
```

其实也然并卵。单单只是实现这个方法，不然报错，Do you know？

那么只剩下这个点了。

```
// Okio$Sink
timeout.throwIfReached();
```
说了半天，我们还没了解Timeout这个类。

```
public class Timeout {
  ...
  private boolean hasDeadline; // 是否设置了超时时间点
  private long deadlineNanoTime; // 超时时间点，纳秒
  private long timeoutNanos; // 超时等待时间，纳秒

  public Timeout() {
  }
  ...
  public void throwIfReached() throws IOException {
    if (Thread.interrupted()) {
      // 判断线程是否中断，若中断抛出异常并清除中断状态
      throw new InterruptedIOException("thread interrupted");
    }

    if (hasDeadline && deadlineNanoTime - System.nanoTime() <= 0) {
      // 设置了超时时间点且超时时间点已经过了，抛异常
      throw new InterruptedIOException("deadline reached");
    }
  }
  ...
}
```

通过刚才那分析，我们的例子永远都不会超时。。。也就是说并没有默认的超时机制。

这里补充一点关于System的nanoTime方法。该方法返回一个精确的纳秒级别的时间戳，但并不同System.currentTimeMillis()是一个相对某个时刻的绝对时间，因此该方法多用于计算时间差。

Timeout还有一个方法是waitUntilNotified，内部用的是Object的wait，再结合方法名，不难猜出是干嘛的。
#### 异步超时
还记得我们篇头说的Socket吗？我们说好的要和超时机制一起讲的，那么来吧。

```
// Okio
public static Sink sink(Socket socket) throws IOException {
    ...
    AsyncTimeout timeout = timeout(socket);
    Sink sink = sink(socket.getOutputStream(), timeout);
    return timeout.sink(sink);
}
```
OK，我们首先还是去了解一下AsyncTimeout。

```
public class AsyncTimeout extends Timeout {
  /** 一次写入的最大字节数 */
  private static final int TIMEOUT_WRITE_SIZE = 64 * 1024;
  ...
  /** 链表头结点 */
  static @Nullable AsyncTimeout head;
  /** 当前结点是否在队列里 */
  private boolean inQueue;
  /** 下一个结点 */
  private @Nullable AsyncTimeout next;
  /** 超时时间点 */
  private long timeoutAt;
  ...
}
```
简单了解了AsyncTimeout的数据结构，然后我们再去看timeout方法。

```
// Okio
private static AsyncTimeout timeout(final Socket socket) {
    return new AsyncTimeout() {
      ...
    };
}
```
行吧，直接new了一个，很粗暴。到此，我们仍然不太了解这个类的作用，因为压根没用到它，那么接下这句应该是关键：

```
// Okio#sink(Socket socket)
timeout.sink(sink)

// AsyncTimeout
public final Sink sink(final Sink sink) {
    return new Sink() {
      @Override public void write(Buffer source, long byteCount) throws IOException {
            ...
            long toWrite = 0L; // 要写入的字节数，不会超过TIMEOUT_WRITE_SIZE
            ...
            boolean throwOnTimeout = false; // 判断要不要抛异常的标志位，默认为false，成功读取为true
            enter(); // 写操作之前执行
            try {
                sink.write(source, toWrite);
                byteCount -= toWrite;
                throwOnTimeout = true;
            } catch (IOException e) {
                throw exit(e); // 写操作异常执行
            } finally {
                exit(throwOnTimeout); // 写操作之后执行
            }
            ...
      }
      ...
    };
}
```

先来看一下enter方法。

```
// AsyncTimeout
public final void enter() {
    if (inQueue) throw new IllegalStateException("Unbalanced enter/exit");
    long timeoutNanos = timeoutNanos();
    boolean hasDeadline = hasDeadline();
    if (timeoutNanos == 0 && !hasDeadline) {
      return; // 默认的情况下，都是直接返回
    }
    // 假设我们的timeout的变量有改动，那么继续往下走
    inQueue = true; // 标识当前AsyncTimeout进入队列
    scheduleTimeout(this, timeoutNanos, hasDeadline);
}
```

OK，接下来就是安排我们的超时规则。

```
// AsyncTimeout
private static synchronized void scheduleTimeout(
      AsyncTimeout node, long timeoutNanos, boolean hasDeadline) {
    // 初始化我们的head，启动Watchdog，这个稍后详解
    if (head == null) {
      head = new AsyncTimeout();
      new Watchdog().start();
    }

    long now = System.nanoTime();
    // 初始化超时时间点
    if (timeoutNanos != 0 && hasDeadline) {
      node.timeoutAt = now + Math.min(timeoutNanos, node.deadlineNanoTime() - now);
    } else if (timeoutNanos != 0) {
      node.timeoutAt = now + timeoutNanos;
    } else if (hasDeadline) {
      node.timeoutAt = node.deadlineNanoTime();
    } else {
      throw new AssertionError();
    }

    // 获取超时时间差
    long remainingNanos = node.remainingNanos(now);
    for (AsyncTimeout prev = head; true; prev = prev.next) {
      if (prev.next == null || remainingNanos < prev.next.remainingNanos(now)) {
        // 插入到超时时间差比自己大的前面，意味着该链表根据时间差排列
        node.next = prev.next;
        prev.next = node;
        if (prev == head) { // 当前AsyncTimeout的前一个是head
          AsyncTimeout.class.notify(); // 唤醒Watchdog
        }
        break;
      }
    }
}
```

这里并没有我们想要的超时规则，那么规则应该是在Watchdog中。

```
private static final class Watchdog extends Thread {
    Watchdog() {
      super("Okio Watchdog");
      setDaemon(true); // 定义线程名字并设为守护线程
    }

    public void run() {
      while (true) {
        try {
          AsyncTimeout timedOut;
          synchronized (AsyncTimeout.class) {
            // 得到AsyncTimeout
            timedOut = awaitTimeout(); 
            // 如果为空，继续下一个
            if (timedOut == null) continue;
            if (timedOut == head) {
              // 如果所有AsyncTimeout都执行完了，那么当前Watchdog的使命也完成了
              head = null;
              return;
            }
          }
          // 超时回调，如关掉Socket，那么输出流在写的时候就会报错即写操作就结束
          timedOut.timedOut();
        } catch (InterruptedException ignored) {
        }
      }
    }
}
```

我们不太明白timedOut是怎么得到的，得到的timedOut代表什么意思，整个逻辑还是较为模糊。

```
static @Nullable AsyncTimeout awaitTimeout() throws InterruptedException {
    // 取到head的下个节点
    AsyncTimeout node = head.next;
    if (node == null) {
      // 如果下个节点为空，也就是说只剩下head了，那么将等待60秒
      long startNanos = System.nanoTime();
      AsyncTimeout.class.wait(IDLE_TIMEOUT_MILLIS);
      // 如果head.next还是为空，也就是没有添加过
      // (System.nanoTime() - startNanos) >= IDLE_TIMEOUT_NANOS 意味着是已超过等待时间
      // 返回head，根据Watchdog的逻辑，将结束自己
      // 反之，说明添加过AsyncTimeout或者说被唤醒了，那么会返回空，即直接走下一个节点
      return head.next == null && (System.nanoTime() - startNanos) >= IDLE_TIMEOUT_NANOS
          ? head  
          : null;
    }
    // 获取node的超时时间差
    long waitNanos = node.remainingNanos(System.nanoTime());
    if (waitNanos > 0) {
      // 时间差大于0意味着该AsyncTimeout还未超时，那么继续等待
      // 等待结束，直接返回空，执行下一个
      long waitMillis = waitNanos / 1000000L;
      waitNanos -= (waitMillis * 1000000L);
      AsyncTimeout.class.wait(waitMillis, (int) waitNanos);
      return null;
    }

    // 如果超时，那么移除已超时的节点并返回
    head.next = node.next;
    node.next = null;
    return node;
}
```

关于进入超时规则以及Watchdog的运行规则我相信你已经搞清楚了，如果还没搞清楚，我真的无能为力了，这么详细的注释。

这里再说一下exit的逻辑，其实比较简单。代码讲解就一起上了。

```
final IOException exit(IOException cause) throws IOException {
    // 如果节点并没有真正的退出，那么直接抛正常异常，反之抛超时的异常
    if (!exit()) return cause; 
    return newTimeoutException(cause);
}

public final boolean exit() {
    if (!inQueue) return false; // 都不在队列，说个毛
    inQueue = false; 
    return cancelScheduledTimeout(this);
}

private static synchronized boolean cancelScheduledTimeout(AsyncTimeout node) {
    // 如果找到节点，那么移除，找到也以为还未发生超时故返回false
    for (AsyncTimeout prev = head; prev != null; prev = prev.next) {
      if (prev.next == node) {
        prev.next = node.next;
        node.next = null;
        return false;
      }
    }
    // 没找到说明是超时的，已经在Watchdog中移除了，返回true
    return true;
}

final void exit(boolean throwOnTimeout) throws IOException {
    boolean timedOut = exit();
    // 如果超时且数据写完了，那么抛一个InterruptedIOException异常
    if (timedOut && throwOnTimeout) throw newTimeoutException(null);
}
```

到这里，超时机制已经讲解完成了。从源码分析，我们知道默认其实并没有超时机制，我们需要自己去制定，但你得了解它的运行机制。

## 图解
### 整体结构

![](/img/okio_data_structure.png)

### 异步超时流程图
粗略的流程图，具体细节最好细读源码讲解。

![](/img/asyn_timeout_flow.png)
## 总结
没啥好说的，square出品，必属精品。关于Okio的优点文中已经总结过了。我非常怀疑square这些大佬是不是都是处女座。比如Retrofit框架，简洁但不简单，牛皮，牛皮。





