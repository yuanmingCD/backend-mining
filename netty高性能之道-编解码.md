#编解码
TCP粘包、拆包

##编解码框架效率对比
性能上主要关注
 1. 编解码效率
 2. 编码后码流大小





##netty编解码框架
netty主要提供四个基础类

 - ByteToMessageDecoder 
 - MessageToMessageDecoder 
 - MessageToByteEncoder
 - MessageToMessageEncoder

1.累加数据 
2.将累加到的数据传递给业务进行业务拆包 
3.清理字节容器 
4.传递业务数据包给业务解码器处理


DelimiterBasedFrameDecoder 消息边界
FixedLengthFrameDecoder 固定长度
LengthFieldBasedFrameDecoder 消息头指定消息长度
LineBasedFrameDecoder  回车换行符



https://github.com/eishay/jvm-serializers/wiki
