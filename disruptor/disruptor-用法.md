# disruptor用法



disruptor是一种生产者-消费者模式，但是disruptor对其中的一些概念进行了包装，并不太好理解，按照我们常用的队列接口进行重新的封装，以展示其用法。



将消息封装成Event

```java
public class MessageEvent
{
    private long value;

    public void set(long value){
        this.value = value;
    }
}
```



Producer





Consumer



