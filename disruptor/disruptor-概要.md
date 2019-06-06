# disruptor 

  



## 背景



加锁

CAS

无需进行上线文切换到内核态，但仍需要处理器必须锁定其指令管道以确保原子性并使用内存屏障使更改对其他线程可见

性能优于



| **Method**                                         | **Time (ms)** |
| -------------------------------------------------- | ------------- |
| Single thread  (单线程)                            | 300           |
| Single thread with lock   (单线程加锁)             | 10,000        |
| Two threads with lock   (两条线程加锁)             | 224,000       |
| Single thread with CAS   (单线程CAS)               | 5,700         |
| Two threads with CAS   (两条线程CAS)               | 30,000        |
| Single thread with volatile write   (单线程可见写) | 4,700         |



### 伪共享



CPU缓存

缓存行







## 设计



