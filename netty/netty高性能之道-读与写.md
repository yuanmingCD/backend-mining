##ByteBuf与ByteBuffer

netty比较核心的类ByteBuf，对java NIO 的ByteBuffer进行了封装



ByteBuf
     +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      |                   |     (CONTENT)    |                  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
读写指针的操作

读 readerIndex增加,写 writerIndex增加 0-writerIndex之间为已写区域，writerIndex-capacity之间为可写区域，0-readerIndex之间为已读区域，即可弃区域，readerIndex-writerIndex之间为为已写未读区域，即可读区域

几个操作
清空 readerIndex = writerIndex = 0

readXXX，writeXXX


读操作，这里是指我们处理请求的过程，首先要从socket中读取数据，那么对于Buffer来说，首先要进行的是写操作。




写操作，即Bytebuf的读操作。
要向外写出数据，首先要构造响应，然后将响应编码，写入到Buffer，最后写出到socket
构造响应与编码过程在这里就省略掉了，我们着重分析一下ByteBuf的分配，以及最后的写出过程。