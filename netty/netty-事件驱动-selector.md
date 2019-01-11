

# Netty-事件驱动-selector

多路复用器

这部分属于jdk的实现，可以通过观察具体实现过程，可以更加深入了解多路复用

```java
public void initSelector() {
        try {
            selector = SelectorProvider.provider().openSelector();
            this.serverChannel1 = ServerSocketChannel.open();
            serverChannel1.configureBlocking(false);
            InetSocketAddress isa = new InetSocketAddress("localhost", this.port1);
            serverChannel1.socket().bind(isa);
            serverChannel1.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
}
```









操作系统的选择



pipe唤醒selector







[Java NIO——Selector机制源码分析系列——转](https://blog.csdn.net/baidu_33116785/article/details/52682629)

[Java NIO类库Selector机制解析（上）](https://blog.csdn.net/haoel/article/details/2224055)

[Java NIO类库Selector机制解析（下）](https://blog.csdn.net/haoel/article/details/2224069)



