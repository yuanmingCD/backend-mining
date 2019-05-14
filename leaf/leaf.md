# leaf

​       Leaf(树叶)是美团点评开源的分布性唯一ID生成，名字来源于“世界上没有两片完全相同的树叶”.目前Leaf覆盖了美团点评公司内部金融、餐饮、外卖、酒店旅游、猫眼电影等众多业务线。在4C8G VM基础上，通过公司RPC方式调用，QPS压测结果近5w/s，TP999 1ms。

## 目录

- 分布式唯一ID
- 源码分析

## 相似产品

[uid-generator](https://github.com/baidu/uid-generator)(百度开源)

## 相关文档

[github地址](https://github.com/Meituan-Dianping/Leaf)

[ leaf 美团分布式ID生成服务](https://tech.meituan.com/MT_Leaf.html)

