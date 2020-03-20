# netty-内存管理-内存池

[TOC]

Netty内存池是将内存的分配管理起来减少内存碎片和避免内存浪费，Netty内存池参考了Slab分配和Buddy分配思想；Slab分配是将内存分割成大小不等的内存块，在用户线程请求时根据请求的内存大小分配最为贴近size的内存快，减少了内存碎片同时避免了内存浪费；Buddy分配是把一块内存块等量分割，回收时候进行合并，尽可能保证系统中有足够大的连续内存；


#### 结构

![内存模型数据结构](images/%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="这里输入图片地址">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">这里输入题注</div>
</center>



内存分级从上到下主要分为：Arena，ChunkList，Chunk，Page，SubPage五级；

整个池化内存通过一个PooledArena数组管理，我们先看下一个PooledArena组件是如何构成

PooledArena

首先看下PooledArena的成员变量以及构造函数

```java
static final boolean HAS_UNSAFE = PlatformDependent.hasUnsafe();

enum SizeClass {
    Tiny,
    Small,
    Normal
}

static final int numTinySubpagePools = 512 >>> 4;

final PooledByteBufAllocator parent;

private final int maxOrder;
final int pageSize;
final int pageShifts;
final int chunkSize;
final int subpageOverflowMask;
final int numSmallSubpagePools;
final int directMemoryCacheAlignment;
final int directMemoryCacheAlignmentMask;
private final PoolSubpage<T>[] tinySubpagePools;
private final PoolSubpage<T>[] smallSubpagePools;

//根据百分比存放的链表
private final PoolChunkList<T> q050;
private final PoolChunkList<T> q025;
private final PoolChunkList<T> q000;
private final PoolChunkList<T> qInit;
private final PoolChunkList<T> q075;
private final PoolChunkList<T> q100;

private final List<PoolChunkListMetric> chunkListMetrics;

// 内存分配、回收指标
private long allocationsNormal;
// We need to use the LongCounter here as this is not guarded via synchronized block.
private final LongCounter allocationsTiny = PlatformDependent.newLongCounter();
private final LongCounter allocationsSmall = PlatformDependent.newLongCounter();
private final LongCounter allocationsHuge = PlatformDependent.newLongCounter();
private final LongCounter activeBytesHuge = PlatformDependent.newLongCounter();

private long deallocationsTiny;
private long deallocationsSmall;
private long deallocationsNormal;

// We need to use the LongCounter here as this is not guarded via synchronized block.
private final LongCounter deallocationsHuge = PlatformDependent.newLongCounter();

// Number of thread caches backed by this arena.
final AtomicInteger numThreadCaches = new AtomicInteger();
```

再看构造函数

```java
protected PoolArena(PooledByteBufAllocator parent, int pageSize,
      int maxOrder, int pageShifts, int chunkSize, int cacheAlignment) {
    this.parent = parent;
    this.pageSize = pageSize;
    this.maxOrder = maxOrder;
    this.pageShifts = pageShifts;
    this.chunkSize = chunkSize;
    directMemoryCacheAlignment = cacheAlignment;
    directMemoryCacheAlignmentMask = cacheAlignment - 1;
    subpageOverflowMask = ~(pageSize - 1);
    tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);
    for (int i = 0; i < tinySubpagePools.length; i ++) {
        tinySubpagePools[i] = newSubpagePoolHead(pageSize);
    }

    numSmallSubpagePools = pageShifts - 9;
    smallSubpagePools = newSubpagePoolArray(numSmallSubpagePools);
    for (int i = 0; i < smallSubpagePools.length; i ++) {
        smallSubpagePools[i] = newSubpagePoolHead(pageSize);
    }

    q100 = new PoolChunkList<T>(this, null, 100, Integer.MAX_VALUE, chunkSize);
    q075 = new PoolChunkList<T>(this, q100, 75, 100, chunkSize);
    q050 = new PoolChunkList<T>(this, q075, 50, 100, chunkSize);
    q025 = new PoolChunkList<T>(this, q050, 25, 75, chunkSize);
    q000 = new PoolChunkList<T>(this, q025, 1, 50, chunkSize);
    qInit = new PoolChunkList<T>(this, q000, Integer.MIN_VALUE, 25, chunkSize);

    q100.prevList(q075);
    q075.prevList(q050);
    q050.prevList(q025);
    q025.prevList(q000);
    q000.prevList(null);
    qInit.prevList(qInit);

    List<PoolChunkListMetric> metrics = new ArrayList<PoolChunkListMetric>(6);
    metrics.add(qInit);
    metrics.add(q000);
    metrics.add(q025);
    metrics.add(q050);
    metrics.add(q075);
    metrics.add(q100);
    chunkListMetrics = Collections.unmodifiableList(metrics);
}
```

结合面PoolArean的结构图我们看一下

- PoolChunkList链表
- PoolChunkList
- PoolChunk 满二叉树，由数组存储
- Page





#### 初始化

#### 内存的分配

分配规则

| 大小               | 分配单位                                                   |
| :----------------- | ---------------------------------------------------------- |
| <512B              | tinySubPagePools                                           |
| 512B~pageSize(4K)  | smallSubPagePools                                          |
| pageSize~chunkSize | PoolChunkList                                              |
| >chunkSize         | 创建非池化的Chunk来分配，并且该Chunk不会放在内存池中重用。 |





1. 对PoolChunk进行初始化, 并将该PoolChunk加入qInit的链中。
   这里有一个细节需要了解下, q050、q025、q000、qInit、q075按照这个顺序排序, 也就是说当在这几个对象都有可分配的内存时, 优先从 q050中分配, 最后从q075中分配。这样安排的考虑是:

- 将PoolChunk分配维持在较高的比例上。
- 保存一些空闲度比较大的内存, 以便大内存的分配。



#### 内存的回收

#### 小结