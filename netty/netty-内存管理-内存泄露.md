# netty-内存管理-内存泄露



引用计数器与可达性分析



java为何会发生内存泄露



最简单的内存泄露实例



netty为何会发生内存泄露

netty buffer内存模型（分三层）

ByteBuf不再使用后（没有引用），没有调用release，导致ByteBuf资源一直被占用无法回收



netty检测内存泄露原理

**原理：**每次创建ByteBuf时，创建一个虚引用对象A指向该ByteBuf对象，如果正常调用release()操作，则虚引用对象A和ByteBuf对象均被回收；如果ByteBuf对象不再使用（没有其他引用）但没有调用release()操作，则GC时虚引用对象A被加入ReferenceQueue中，通过判断队列是否为空，即可知道是否存在内存泄露。



## 什么是内存泄露



## 为什么会内存泄漏





## 如何检测内存泄露





[netty 堆外内存泄露排查盛宴](https://www.jianshu.com/p/4e96beb37935?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

[Netty 防止内存泄漏措施](https://www.infoq.cn/article/olLlvGFx*Kr0UV9K7tez)