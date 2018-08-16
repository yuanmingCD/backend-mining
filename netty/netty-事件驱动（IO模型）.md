# 事件驱动

提到IO模型,Unix网络编程的五种模型，但实际上这个分类不是很好理解，本文进行重新分类


同步与异步
>可以理解为客户端的动作，简单理解为客户端主动轮询即为同步，客户端被动监听即为异步


阻塞与非阻塞
>可以理解为服务端的动作，阻塞，服务端不会立即返回结果，表现为线程挂起等待唤醒，等到有数据时在返回，非阻塞，服务端不管有没有数据会立即返回，线程去处理其它事物。



### BIO（Blocking IO ，阻塞IO）



 accept，read，write会发生阻塞，直到数据到达或异常才会返回，一个线程只能处理一个请求，线程的资源有限，而且频繁切换会导致系统资源消耗

### NIO（Non-blocking/New IO，非阻塞IO）

事件注册之后会立即返回，这样就可以去做别的事，但是要定时轮询是否有事件就绪来处理事件，显然同一时间处理能力变强。

### AIO（Asynchronous IO,NIO 2.0, 异步IO ）

 除了阻塞和非阻塞的概念外，往往还会出现同步和异步的概念。如果BIO和NIO是阻塞和非阻塞的区别，那么AIO与NIO就是同步和异步的区别。
 同步简单理解为仍需主动轮询去获取
 异步可以简单理解为通过注册回调函数

  Linux系统中的信号机制提供了一直通知某事情发生的方法。异步IO通知内核某个操作，并让内核在整个操作完成后通知应用程序
  注册回调函数，触发IO事件时执行回调通知应用程序，这样就连轮询的过程也省略了，极大提高了性能
##多路复用（Multiplexing，多工，事件驱动）
多路复用可以理解为NIO的多线程实现，将多个socket注册在一个多路复用器上，多路复用是Reactor线程的核心，后文会具体阐述
多路复用实现select，pselect，poll，epoll，kqueue

###select
指定感兴趣的描述符集合（即OP_READ,OP_WRITE,OP_ACCEPT,OP_CONNECT），遍历当前进程socket事件，查找就绪事件
缺点： 

 1. 每次调用select，需要把fd_set从用户空间和内核空间之间进行拷贝，fd很多时开销很大
 2. 每次调用select都要在内核遍历进程的所有fd，fd很多时开销很大，随着fd增长而线性增长
 3. select支持的文件描述符有限，默认是1024，太小


###poll
poll的实现和select非常相似，只是文件描述符fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构; poll不是为每个条件（读/写/异常）构造一个描述符集，而是构造一个pollfd结构，每个数组元素指定一个描述符编号以及我们对该描述符感兴趣的条件

poll和select同样存在一个缺点：包含大量文件描述符的数组被整体复制于用户态和内核的地址空间自己，无论是否就绪，开销随着文件描述符数量增加而增加。

###epoll
而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait:

epoll_create是创建一个epoll句柄 epollfd；
epoll_ctl是注册要监听的事件类型；
epoll_wait则是等待事件的产生。
针对fd_set每次从用户空间和内核空间之间进行拷贝的缺点，epoll在 epoll_ctl函数中，每次在注册fd到epoll_create生成的epoll句柄时，把fd拷贝进内核，而不是在 epoll_wait的时候重复拷贝。因此，epoll保证每个fd只拷贝一次，在循环的epoll_wait不进行拷贝，而select和poll在循环中每次都需要拷贝。

针对select/poll在内核中遍历所有fd，epoll不会每次都将current（用户写select的线程）轮流加入fd对应设备的等待队列，而是在 epoll_ctl时把current挂一遍，并为每个fd指定一个回调函数，设备就绪唤醒等待队列上的线程时，会调用回调函数；回调函数把就绪的fd放入一个就绪链表。
epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（ schedule_timeout() 实现睡一会判断一会）

针对fd大小的限制，epoll没有限制，支持的fd上限时最大可以打开文件的数目，一般远大于2048，如1GB机器可打开的是10万左右 cat /proc/sys/fs/file-max可查看

epoll支持水平触发（level trigger，LT）和边缘触发（edge trigger，ET
）两种模式
>LT(level triggered) 是默认/缺省的工作方式，同时支持 block和no_block socket。这种工作方式下，内核会通知你一个fd是否就绪，然后才可以对这个就绪的fd进行I/O操作。就算你没有任何操作，系统还是会继续提示fd已经就绪，不过这种工作方式出错会比较小，传统的select/poll就是这种工作方式的代表。

>ET(edge-triggered) 是高速工作方式，仅支持no_block socket，这种工作方式下，当fd从未就绪变为就绪时，内核会通知fd已经就绪，并且内核认为你知道该fd已经就绪，不会再次通知了，除非因为某些操作导致fd就绪状态发生变化。如果一直不对这个fd进行I/O操作，导致fd变为未就绪时，内核同样不会发送更多的通知，因为only once。所以这种方式下，出错率比较高，需要增加一些检测程序。

LT可以理解为水平触发，只要有数据可以读，不管怎样都会通知。而ET为边缘触发，只有状态发生变化时才会通知，可以理解为电平变化。


不管是哪种I/O机制，都无法避免fd在操作过程中拷贝的问题，而epoll使用了mmap(是指文件/对象的内存映射，被映射到多个内存页上)，所以同一块内存就可以避免这个问题。

只有，基于内核2.6版本之后的linux系统支持
###kqueue
BSD



select与poll比较

epoll与kqueue
性能
从性能角度讲，epoll存在一个设计上的缺陷；它不能在单次系统调用中多次更新兴趣集。当你的兴趣集中有100个文件描述符需要更新状态时，你不得不调用100次epoll_ctl()函数。性能降级在过渡的系统调用时表现的非常明显，这篇文章有做解释。我猜这是Banga et al原来工作的遗留，正如declare_interest()只支持一次调用一次更新那样。相对的，你可以在一次的kevent调用中指定进行多次兴趣集更新。
在kqueue中，kevent结构体支持多种非文件事件，包括 socket、信号、定时器、AIO、VNODE、PIPE

再从读取数据的过程看这几个实现的区别
| 过程 | select&poll  | epoll&kqueue |ICOP|
| :------------ |:---------------:| -----:|-----:|
| 注册感兴趣事件| 1024&无限制| 无限制 | 无限制|
|获取就绪事件集 |主动轮询|   异步回调 |异步回调|
| 处理就绪事件 |用户程序处理| 用户程序处理 |系统处理|
| 应用程序处理| 用户程序处理|用户程序处理 |用户程序处理 |
netty选择哪种多路复用？
由于多路复用与系统调用，这个与操作系统有关，linux内核2.6版本之后的开始支持epoll调用，而mac os则为Kquue http://netty.io/wiki/native-transports.html
jdk在启动的时候会根据操作系统自动选择选用哪种多路复用的实现
 - macosx: KQueueSelectorProvider
 - solaris: DevPollSelectorProvider
 - Linux: EPollSelectorProvider (Linux kernels >= 2.6)或 PollSelectorProvider
 - windows: WindowsSelectorProvider

 相比于jdk实现，netty的实现有如下优势
 - Netty的 epoll transport使用 epoll edge-triggered 而 java的 nio 使用
   level-triggered. 
 - netty epoll transport 暴露了更多的nio没有的配置参数， 如
   TCP_CORK, SO_REUSEADDR等等
   



java nio solector空轮询bug
Netty的解决办法

对Selector的select操作周期进行统计，每完成一次空的select操作进行一次计数，

若在某个周期内连续发生N次空轮询，则触发了epoll死循环bug。

重建Selector，判断是否是其他线程发起的重建请求，若不是则将原SocketChannel从旧的Selector上去除注册，重新注册到新的Selector上，并将原来的Selector关闭。
