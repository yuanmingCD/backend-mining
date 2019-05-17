# leaf-源码剖析



leaf提供两种发号模式

- segment号段模式(数据库自增ID)
- snowflake模式

Leaf代码量并不大，结构也很清晰，这里就不对代码结构进行分析了，核心代码在leaf-core中，我们结合源码来看下具体两种模式的实现

## segment模式

这种模式就是基于数据库实现，核心代码都在SegmentIDGenImpl类中。

### 数据库

数据库表leaf_alloc：

```sql
+-------------+--------------+------+-----+-------------------+-----------------------------+
| Field       | Type         | Null | Key | Default           | Extra                       |
+-------------+--------------+------+-----+-------------------+-----------------------------+
| biz_tag     | varchar(128) | NO   | PRI |                   |                             |
| max_id      | bigint(20)   | NO   |     | 1                 |                             |
| step        | int(11)      | NO   |     | NULL              |                             |
| desc        | varchar(256) | YES  |     | NULL              |                             |
| update_time | timestamp    | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------------+--------------+------+-----+-------------------+-----------------------------+
```

涉及到的几个数据库操作

```sql
--获取当前所有标签的最大值，用于服务启动时初始化
SELECT biz_tag, max_id, step, update_time FROM leaf_alloc
--获取某一标签下最大值
SELECT biz_tag, max_id, step FROM leaf_alloc WHERE biz_tag = #{tag}
--批量获取ID，并按照默认配置，更新数据库
UPDATE leaf_alloc SET max_id = max_id + step WHERE biz_tag = #{tag}
--批量获取ID，并自定义更新数据库
UPDATE leaf_alloc SET max_id = max_id + #{step} WHERE biz_tag = #{key}
--获取当前所有标签
SELECT biz_tag FROM leaf_alloc
```

从SQL中可见，leaf在实现上并未采用数据库自增ID实现的发号，而是记录最大ID，在服务中实现自增发号。

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

    private volatile int step;//与mysql实例数相等，用于扩展数据库资源
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

对于其中涉及的几个参数做简要介绍，为进一步提高性能，SegmentBuffer实现了双buffer的优化，这部分代码逻辑理封装在SegmentBuffer中会更优雅，但是目前的代码是放在了发号逻辑中了，那就在后文的发号逻辑中分析。

- step，实际上就是分片的思想，主要用于数据库实例的横向扩展，step值与数据库实例相同，自增步长也为step值。leaf并非通过这种方式实现，而是是通过记录最大ID，在程序中实现自增，这里只是介绍一种思路。，可通过设置表自增配置实现：

  ```sql
  set global auto_increment_increment=2;
  set global auto_increment_offset=2;
  ```

  但是这种方式并不具备动态扩展的能力，

  - 实例加入，可以将新实例设置的auto_increment_offset值远大于当前ID，然后逐条修改之前已存在实例的配置，auto_increment_increment和auto_increment_offset两个参数都需要修改，同样保证正在修改的实例的auto_increment_offset值要大于当前ID
  - 实例摘除，可以忽略不管，会有一部分ID值获取不到，造成一定的浪费。

  leaf的实现扩展与此类似，只不过相比修改配置而言，直接修改字段值更加方便易操作。

  初始化

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

接下来看具体如何实现初始化操作，以及ID获取。

**buffer初始化**

```java
public void updateSegmentFromDb(String key, Segment segment) {
  StopWatch sw = new Slf4JStopWatch();
  SegmentBuffer buffer = segment.getBuffer();
  LeafAlloc leafAlloc;
 //1.buffer未初始化的先初始化
  if (!buffer.isInitOk()) {
    leafAlloc = dao.updateMaxIdAndGetLeafAlloc(key);
    buffer.setStep(leafAlloc.getStep());
    buffer.setMinStep(leafAlloc.getStep());//leafAlloc中的step为DB中的step
  } else if (buffer.getUpdateTimestamp() == 0) {
    leafAlloc = dao.updateMaxIdAndGetLeafAlloc(key);
    buffer.setUpdateTimestamp(System.currentTimeMillis());
    buffer.setMinStep(leafAlloc.getStep());//leafAlloc中的step为DB中的step
  } else {
    //已初始化完毕，正常提供发号功能
    long duration = System.currentTimeMillis() - buffer.getUpdateTimestamp();
    int nextStep = buffer.getStep();
    //SEGMENT_DURATION 定值，15分钟；MAX_STEP 定值，1000000
    if (duration < SEGMENT_DURATION) {
      //下次获取步长如果小于最大步长1000000的一半，就扩大一倍，保证nextStep在50w到100w之间
      if (nextStep * 2 > MAX_STEP) {
        //do nothing
      } else {
        nextStep = nextStep * 2;
      }
    //距上一次获取时间小于7.5分钟不做处理，调整步长
    } else if (duration < SEGMENT_DURATION * 2) {
      //do nothing with nextStep
    } else {
      //距上一次获取时间在7.5-15分钟之间，调整步长
      nextStep = nextStep / 2 >= buffer.getMinStep() ? nextStep / 2 : nextStep;
    }
    logger.info("leafKey[{}], step[{}], duration[{}mins], nextStep[{}]", key, buffer.getStep(), String.format("%.2f",((double)duration / (1000 * 60))), nextStep);
    LeafAlloc temp = new LeafAlloc();
    temp.setKey(key);
    temp.setStep(nextStep);
    //从数据库获取ID
    leafAlloc = dao.updateMaxIdByCustomStepAndGetLeafAlloc(temp);
    //更新buffer
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

主要功能是，根据本次更新距上次更新的时间差以及消耗的ID值，动态调整本次获取的ID值大小。

**ID获取**

```java
public Result getIdFromSegmentBuffer(final SegmentBuffer buffer) {
    while (true) {
        try {
            //buffer读锁加锁，防止重复获取
            buffer.rLock().lock();
            final Segment segment = buffer.getCurrent();
           //获取ID的时候检查当前Segment状态，是否需要向buffer中异步补充ID资源
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
                            //另一个buffer更新完成，设置标志位
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
            //正常获取到id
            long value = segment.getValue().getAndIncrement();
            if (value < segment.getMax()) {
                return new Result(value, Status.SUCCESS);
            }
        } finally {
            buffer.rLock().unlock();
        }
        //若没有获取到,等待buffer从数据库更新完成
        waitAndSleep(buffer);
        try {
            //对buffer进行修改，加写锁
            buffer.wLock().lock();
            final Segment segment = buffer.getCurrent();
            long value = segment.getValue().getAndIncrement();
            if (value < segment.getMax()) {
                return new Result(value, Status.SUCCESS);
            }
            //切换buffer
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

1. 在获取ID的时候判断当前Segment资源是否充足，不足资源的90%，开启异步线程对另一个buffer进行填充
2. 线程内调用之前描述的方法，获取ID并更新buffer切换就绪标志位
3. 当前资源不足时，进入下面的try-catch代码，完成buffer切换。

可见双buffer是对读写分离的优化，减少锁冲突，是一种比较通用的优化思想。

落地优化

### 总结

Leaf-segment，主要通过以下方法实现优化。

- 分步长实现数据库实例横向扩展
- 批量获取，减少数据库IO
- 异步获取并缓存，提高性能
- 双buffer优化，减少读写冲突



## snowflake模式

核心代码在SnowflakeIDGenImpl中

- 机器id分配
- 时钟回拨





通过之前对snowflake的分析可以看出，需要分配机器唯一ID，通过zookper实现。



