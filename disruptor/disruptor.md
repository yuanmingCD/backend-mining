# Disruptor——高性能单机队列

定位： 高性能单机队列

> High performance alternative to bounded queues for exchanging data between concurrent threads
>
> 





需要注意的是





主要知识点：

- [**Ring Buffer**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/RingBuffer.java): The Ring Buffer is often considered the main aspect of the Disruptor, however from 3.0 onwards the Ring Buffer is only responsible for the storing and updating of the data (Events) that move through the Disruptor. And for some advanced use cases can be completely replaced by the user.
- [**Sequence**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/Sequence.java): The Disruptor uses Sequences as a means to identify where a particular component is up to. Each consumer (EventProcessor) maintains a Sequence as does the Disruptor itself. The majority of the concurrent code relies on the the movement of these Sequence values, hence the Sequence supports many of the current features of an AtomicLong. In fact the only real difference between the 2 is that the Sequence contains additional functionality to prevent false sharing between Sequences and other values.
- [**Sequencer**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/Sequencer.java): The Sequencer is the real core of the Disruptor. The 2 implementations (single producer, multi producer) of this interface implement all of the concurrent algorithms use for fast, correct passing of data between producers and consumers.
- [**Sequence Barrier**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/SequenceBarrier.java): The Sequence Barrier is produced by the Sequencer and contains references to the main published Sequence from the Sequencer and the Sequences of any dependent consumer. It contains the logic to determine if there are any events available for the consumer to process.
- [**Wait Strategy**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/WaitStrategy.java): The Wait Strategy determines how a consumer will wait for events to be placed into the Disruptor by a producer. More details are available in the section about being optionally lock-free.
- **Event**: The unit of data passed from producer to consumer. There is no specific code representation of the Event as it defined entirely by the user.
- [**EventProcessor**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/EventProcessor.java): The main event loop for handling events from the Disruptor and has ownership of consumer's Sequence. There is a single representation called [BatchEventProcessor](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/BatchEventProcessor.java) that contains an efficient implementation of the event loop and will call back onto a used supplied implementation of the EventHandler interface.
- [**EventHandler**](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/EventHandler.java): An interface that is implemented by the user and represents a consumer for the Disruptor.
- **Producer**: This is the user code that calls the Disruptor to enqueue Events. This concept also has no representation in the code.



## Github地址

https://github.com/LMAX-Exchange/disruptor

使用

## 相关文档

[高性能队列——Disruptor](https://tech.meituan.com/disruptor.html)

[Java与CPU缓存是如何亲密接触的](https://mp.weixin.qq.com/s/ODJqoiHYwAhRCMnVjunsbQ)

[你应该知道的高性能无锁队列Disruptor](https://juejin.im/post/5b5f10d65188251ad06b78e3?utm_source=gold_browser_extension)