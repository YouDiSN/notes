#REDIS源码26-Sentinel

文档如下

[官方Sentinel文档](https://redis.io/topics/sentinel)

下面就写一点阅读官方文档的体会吧，源码实在搞不动了。。。

- Redis的Sentinel部署：一主两从三个sentinel应该是一个基本的标配。配置成基数个主要也是需要大部分能同意。就和其他的集群类似的，要求部署基数个。
- 当一个Master和两个Slave失去联系的时候（比如网络分区故障），这个时候Client1可能还在往Master写数据。当然这些数据基本上不可能回复了，所以Redis提供了一个配置，如果Master挂了，就不再接受Client的数据写入，这样Client这次写失败就会通过配置的Sentinel节点找到其他存活的集群节点，然后进行重试。
- 如果真的只有两个Redis节点，一主一从，还可以把Sentinel部署在Client端，这样来确保不会因为两个而出问题。
- 

