[Disruptor](https://github.com/LMAX-Exchange/disruptor)是一款开源的低延迟，高吞吐量的有界内存队列，
也在各种类型的开源框架的实现中被采用，比如Spring、HBase、Storm、Log4j等等。

Disruptor不是分布式组件，而是在进程内的一个高性能队列，所以Disruptor的设计无需考虑CAP和通信机制等相关的内容，
而是只专注于性能效率。

这篇文章主要聊一聊Disruptor中的一些实现高性能的手段以及它的设计思路，如何抽象出我们可以借鉴的思想。

[1. 更高效的内存利用]()
[2. 提升CPU缓存利用率]()
[3. CAS机制以及生产 & 消费流程]()
[4. Disruptor主要组件]()
[5. 生产 & 消费流程]()
[6. 总结]()

---

Disruptor的高性能的手段主要在如下几个方面：

### 1. 更高效的内存利用

首先内存中数组数据结构是比链表更具有空间局部性，关于[局部性原理]()内容有在另一篇中讨论过。
另外在内存分配时，RingBuffer采用环形数组结构，在初始化时候一次性创建并且数据元素在整个过程中循环利用，
避免不必要的再次开辟内存空间和频繁的GC回收。另外数据的内存连续的特性对CPU的分支预测也非常友好。

另外对于数组的大小RingBuffer和JDK中HashMap内置的扩容一样，都是2的整数次幂为标准（2^n的特点是二进制形式的值-1后各个位都为1），
是因为在计算index时候有进行'&'的位运算操作，这种计算方式比较快。在HashMap中可以减少hash冲突从而提升hash()的效率，
而在RingBuffer中更重要，如果不按照2的整数次幂初始化，可能会出现数组中的内存浪费。

所以RingBuffer在构造的时候以异常来强制要求size的标准。

```
public AbstractSequencer(int bufferSize, WaitStrategy waitStrategy) {
    ...
    if (Integer.bitCount(bufferSize) != 1) {
        throw new IllegalArgumentException("bufferSize must be a power of 2");
    }
    ...
}
```

RingBuffer就算index的方式：array index = sequence & （array length－1）；

---

### 2. 提升CPU缓存利用率

这部分的实现是Disruptor中比较精彩的，实现后我们再总结起来可能比较容易理解，
但是从零到一的实现是需要悉知硬件层面的原理并且能够再加以利用的。

我们知道内存可以做硬盘的缓存，CPU缓存可以做内存的缓存，空间容量上越来越小的同时访问速速也越来越快，
并且每一层都是相差了几个数量级。内存的访问速度相比于CPU的缓存是要慢上几十甚至一百倍的，
当然存储器的造假成本也有很大的差异。

在操作系统内核中地址空间是所有进程共享的，所以内核空间的全局变量是任何进程都可以访问的，
而cpu的多级缓存策略中，是以缓存行的机制来进行加载的，缓存行的大小通常是64字节。
如果程序中有8个连续的Long类型数据，程序中访问一个Long的变量的时候，CPU会同时加载了另外7个数据（在这个数据前后定义的变量）。
这对数据这种数据结构来说可能会得到从缓存中很快访问的效果。
但是同时这也会带来一些问题称之为"伪共享"，由于共享缓存行导致缓存失效，
**其实本质上还是一个数据冗余带来的数据一致性的问题**，
这主要也取决于CPU缓存保证数据一致性遵循的（[MESI协议](https://zh.wikipedia.org/wiki/MESI%E5%8D%8F%E8%AE%AE)）。

简单说就是说当两个线程T1,T2操作独立的两个数，而这两个数又是布局连续的处在同一个缓存行中，
这时候可能会出现T2要重新读取当前操作的数据的时候，
却发现需要重新去内存里拉取新值，**因为T1在操作它的数据的时候，是以一个缓存行作为一个写回单位的。**
这时候读取操作自然就大大变慢了，需要从主存中重新拉去数据，这是CPU遵循协议所规定的做法，
但在这里的场景确实是不必要且低效的。

Disruptor能够在应用层面做到更高效的利用CPU缓存，是怎么做的呢？

其实是在将变量所在缓存行做了隔离，方法是用一些无意义且不会再进行修改的填充变量，也是一种以空间换时间的实践。

```
class LhsPadding
{
    // 前面用来补齐缓存行的变量
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding
{
    // 真正要操作的变量
    protected volatile long value;
}

class RhsPadding extends Value
{
    // 后面用来补齐缓存行的变量
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```


---

### 3. CAS机制应用

CAS机制在JDK中的应用并不罕见，无论内置的锁还是显示的锁很多都有着CAS操作，
准确说CAS更是一种的思想，也是可以直接在应用程序中体现的，

下面是在 com.lmax.disruptor.MultiProducerSequencer 多生产者的实现中生产者入队过程的核心代码，
主要流程是如果没有足够的空余位置，就用 LockSupport.parkNanos(1) 出让CPU的使用权，
避免当前出现线程总是空转，如果有的话CAS方式抢占位置。
```
...

do {
  // cursor类似于入队索引，指的是上次生产到的位置
  current = cursor.get();
  // 增量生产传入的个数
  next = current + n;
  // 减掉一圈循环，用来判断消费者是否超过了消费者
  long wrapPoint = next - bufferSize;
  // 获取上一次的最小消费位置
  long cachedGatingSequence = gatingSequenceCache.get();
  // 如果没有足够的空余位置
  if (wrapPoint>cachedGatingSequence || cachedGatingSequence>current){
    // 有些消费者消费的可能比较慢，生产者就必须等待最慢的消费者，重新计算所有消费者里面的最小值位置
    long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
    // 生产者已经从后面追过消费者，所以此时仍然没有满足的空余位置，出让CPU，再执行下一次循环尝试
    if (wrapPoint > gatingSequence){
      LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
      continue;
    }
    // 从新设置上一次的最小消费位置
    gatingSequenceCache.set(gatingSequence);
  } else if (cursor.compareAndSet(current, next)){
    // CAS获取写入位置成功
    break;
  }
} while (true);

```
上面是生产者对新Event获得一个"槽位"的主要流程，后面还有写avaliableBuffer的步骤以及 publish(sequence) 进行提交，

---

### 4. Disruptor主要组件

---

### 5. 总结


