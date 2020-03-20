# netty-内存管理

池化与非池化
所谓池化，就是预留一些资源，池化优势

 1. 资源可控，更加安全
 2. 减小创建和销毁的资源开销，性能更好。

我们日常接触到的池化技术有很多，线程池、连接池、对象池、内存池等，线程池与连接池并非本文的重点，这里着重介绍netty的对象池和内存池。

netty的内存管理基于



Recycler首先从当前线程绑定的值中获取stack，我们可以得知Netty中其实是每个线程关联着一个对象池，直接关联对象为Stack，先看看池中是否有可用对象，如果有则直接返回，如果没有则新创建一个Handle，并且调用newObject来新创建一个对象并且放入Handler的value中，newObject由用户自己实现。







## 参考文章

[Netty源码分析（八）—内存池分析](https://blog.csdn.net/u013967175/article/details/78627801)

https://my.oschina.net/andylucc/blog/614589

https://blog.csdn.net/u013967175/article/details/78627769

https://juejin.im/post/5e09cac6f265da33e97fe485?utm_source=gold_browser_extension

