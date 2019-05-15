# leaf



leaf提供两种发号模式

- segment号段模式(数据库自增ID)
- snowflake模式

Leaf代码量并不大，结构也很清晰，这里就不对代码结构进行分析了，核心代码在leaf-core中，我们结合源码来看下具体两种模式的实现

## segment模式

这种模式就是基于数据库自增ID实现

双buffer优化

这是leaf做的一个优化，实际上就是将读写进行分离，

核心代码都在SegmentIDGenImpl类中。

### 组件

首先需要了解一个比较核心的数据结构SegmentBuffer

```java
public class SegmentBuffer{
    private String key;
    private Segment[] segments; //双buffer
    private volatile int currentPos; //当前的使用的segment的index
    private volatile boolean nextReady; //下一个segment是否处于可切换状态
    private volatile boolean initOk; //是否初始化完成
    private final AtomicBoolean threadRunning; //线程是否在运行中
    private final ReadWriteLock lock;

    private volatile int step;
    private volatile int minStep;
    private volatile long updateTimestamp;

    public SegmentBuffer() {
        segments = new Segment[]{new Segment(this), new Segment(this)};
        currentPos = 0;
        nextReady = false;
        initOk = false;
        threadRunning = new AtomicBoolean(false);
        lock = new ReentrantReadWriteLock();
    }
}
```

SegmentBuffer是直接获取ID的缓存，因为获取ID实际上是数据库操作，若每次获取都进行一次数据库操作显然数据库的压力会比较大，因而进行预先对数据库进行批量操作，批量获取ID，存储在SegmentBuffer，待获取时直接内存操作返回。

- **更加稳定**，减少数据库查询带来的请求毛刺。
- **性能更好**，请求链路更短，直接内存返回。
- **容灾**，内存中缓存一定数量的ID，可以在数据库不可用时，在一段时间内继续提供的服务。

为进一步提高性能，SegmentBuffer实现了双buffer的优化，这部分代码逻辑理封装在SegmentBuffer中会更优雅，但是目前的代码是放在了发号逻辑中了，那就在后文的发号逻辑中分析。

### 初始化

```java
public boolean init() {
        logger.info("Init ...");
        // 确保加载到kv后才初始化成功
        updateCacheFromDb();
        initOK = true;
        updateCacheFromDbAtEveryMinute();
        return initOK;
}
```

1. 根据数据库更新好缓存信息
2. 更新标志位；
3. 开启定时更新缓存





### 发号

```java
public Result get(final String key) {
  if (!initOK) {
    return new Result(EXCEPTION_ID_IDCACHE_INIT_FALSE, Status.EXCEPTION);
  }
  if (cache.containsKey(key)) {
    SegmentBuffer buffer = cache.get(key);
    if (!buffer.isInitOk()) {
      synchronized (buffer) {
        if (!buffer.isInitOk()) {
          try {
            updateSegmentFromDb(key, buffer.getCurrent());
            logger.info("Init buffer. Update leafkey {} {} from db", key, buffer.getCurrent());
            buffer.setInitOk(true);
          } catch (Exception e) {
            logger.warn("Init buffer {} exception", buffer.getCurrent(), e);
          }
        }
      }
    }
    return getIdFromSegmentBuffer(cache.get(key));
  }
  return new Result(EXCEPTION_ID_KEY_NOT_EXISTS, Status.EXCEPTION);
}
```

过程

1. 判断是否初始化完成，未完成直接抛异常
2. 判断缓存中是否有key相关的配置，没有直接抛异常。这个key实际上就是为不同的业务隔离产生独立的ID体系
3. 判断buffer是否初始化，未初始化调用updateSegmentFromDb方法进行初始化，已初始化调用getIdFromSegmentBuffer方法获取ID

接下来看具体如何实现初始化操作，以及ID获取

**buffer初始化**

```java
public void updateSegmentFromDb(String key, Segment segment) {
  StopWatch sw = new Slf4JStopWatch();
  SegmentBuffer buffer = segment.getBuffer();
  LeafAlloc leafAlloc;
  if (!buffer.isInitOk()) {
    leafAlloc = dao.updateMaxIdAndGetLeafAlloc(key);
    buffer.setStep(leafAlloc.getStep());
    buffer.setMinStep(leafAlloc.getStep());//leafAlloc中的step为DB中的step
  } else if (buffer.getUpdateTimestamp() == 0) {
    leafAlloc = dao.updateMaxIdAndGetLeafAlloc(key);
    buffer.setUpdateTimestamp(System.currentTimeMillis());
    buffer.setMinStep(leafAlloc.getStep());//leafAlloc中的step为DB中的step
  } else {
    long duration = System.currentTimeMillis() - buffer.getUpdateTimestamp();
    int nextStep = buffer.getStep();
    if (duration < SEGMENT_DURATION) {
      if (nextStep * 2 > MAX_STEP) {
        //do nothing
      } else {
        nextStep = nextStep * 2;
      }
    } else if (duration < SEGMENT_DURATION * 2) {
      //do nothing with nextStep
    } else {
      nextStep = nextStep / 2 >= buffer.getMinStep() ? nextStep / 2 : nextStep;
    }
    logger.info("leafKey[{}], step[{}], duration[{}mins], nextStep[{}]", key, buffer.getStep(), String.format("%.2f",((double)duration / (1000 * 60))), nextStep);
    LeafAlloc temp = new LeafAlloc();
    temp.setKey(key);
    temp.setStep(nextStep);
    leafAlloc = dao.updateMaxIdByCustomStepAndGetLeafAlloc(temp);
    buffer.setUpdateTimestamp(System.currentTimeMillis());
    buffer.setStep(nextStep);
    buffer.setMinStep(leafAlloc.getStep());//leafAlloc的step为DB中的step
  }
  // must set value before set max
  long value = leafAlloc.getMaxId() - buffer.getStep();
  segment.getValue().set(value);
  segment.setMax(leafAlloc.getMaxId());
  segment.setStep(buffer.getStep());
  sw.stop("updateSegmentFromDb", key + " " + segment);
}
```



**ID获取**

```java
public Result getIdFromSegmentBuffer(final SegmentBuffer buffer) {
    while (true) {
        try {
            buffer.rLock().lock();
            final Segment segment = buffer.getCurrent();
            if (!buffer.isNextReady() && (segment.getIdle() < 0.9 * segment.getStep()) && buffer.getThreadRunning().compareAndSet(false, true)) {
                service.execute(new Runnable() {
                    @Override
                    public void run() {
                        Segment next = buffer.getSegments()[buffer.nextPos()];
                        boolean updateOk = false;
                        try {
                            updateSegmentFromDb(buffer.getKey(), next);
                            updateOk = true;
                            logger.info("update segment {} from db {}", buffer.getKey(), next);
                        } catch (Exception e) {
                            logger.warn(buffer.getKey() + " updateSegmentFromDb exception", e);
                        } finally {
                            if (updateOk) {
                                buffer.wLock().lock();
                                buffer.setNextReady(true);
                                buffer.getThreadRunning().set(false);
                                buffer.wLock().unlock();
                            } else {
                                buffer.getThreadRunning().set(false);
                            }
                        }
                    }
                });
            }
            long value = segment.getValue().getAndIncrement();
            if (value < segment.getMax()) {
                return new Result(value, Status.SUCCESS);
            }
        } finally {
            buffer.rLock().unlock();
        }
        waitAndSleep(buffer);
        try {
            buffer.wLock().lock();
            final Segment segment = buffer.getCurrent();
            long value = segment.getValue().getAndIncrement();
            if (value < segment.getMax()) {
                return new Result(value, Status.SUCCESS);
            }
            if (buffer.isNextReady()) {
                buffer.switchPos();
                buffer.setNextReady(false);
            } else {
                logger.error("Both two segments in {} are not ready!", buffer);
                return new Result(EXCEPTION_ID_TWO_SEGMENTS_ARE_NULL, Status.EXCEPTION);
            }
        } finally {
            buffer.wLock().unlock();
        }
    }
}
```

落地优化

### 总结

- 双buffer优化



## snowflake模式





