# Zookeeper会话管理



## 客户端

初次连接



重连

在客户端与服务器断开连接后，会抛出CONNECTION_LOSS异常，Zookeeper 客户端会自动从地址列表重新逐个选取新的地址并尝试进行重新连接，直到最终成功连接上 Zookeeper 集群中的一台机器。在这种情况下，再次连接上服务端的客户端可能会处于以下两种状态之一。

- **CONNECTED**：如果在会话超时时间内重新连接上 Zookeeper 集群中任意一台机器，那么被视为重连成功。
- **EXPIRED**：如果是在会话超时时间以外重新连接上，那么服务端其实已经对该会话进行了会话清理操作，因此在此连接上的会话将被视为非法会话，出现SESSION_EXPIRED异常



心跳检测

如果客户端发现在sessionTimeout/3时间内尚未和服务器进行过任何通信，即没有向服务端发送任何请求，那么就会主动发起一个PING请求。

## 服务端







会话

session，zookeeper客户端和服务端之间的一个tcp连接，

- SessionId：会话ID，唯一标识一个会话，每次客户端创建新会话的时候，Zookeeper 都会为其分配一个全局唯一的 SessionId。
- TimeOut：会话超时时间。客户端在构造 Zookeeper 实例的时候，会配置一个 sessionTimeOut 参数用于指定会话的超时时间。服务器会根据自己的超时时间限制最终确定会话的超时时间。
- TickTime：下次会话超时时间点。为了便于对会话实行 “分桶策略” 管理，同时也是为了高效低耗地实现会话的超时检查与清理，Zookeeper 会为每个会话标记一个下次会话超时时间点。TickTime是一个13位的 long 型数据，其值接近于当前时间加上 TimeOut，但不完全相等。
- isClosing：用于标记一个会话是否已经被关闭。通常当服务端检测到一个会话已经超时失效的时候，会将该属性标记为 “已关闭”，这样就能确保不再处理来自该会话的请求。



分桶策略

基于时间的分桶实现，主要为SessionTracker组件





会话迁移



会话激活

服务端会接收客户端心跳，进行会话激活，重置超时时间

- 只要客户端向服务端发送请求，包括读或写请求，那么就会触发一次会话激活。
- 如果客户端发现在sessionTimeout/3时间内尚未和服务器进行过任何通信，即没有向服务端发送任何请求，那么就会主动发起一个PING请求，服务端收到该请求后，就会触发上述第一种情况下的会话激活。



会话清理