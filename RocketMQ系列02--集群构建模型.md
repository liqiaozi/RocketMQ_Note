# 0.前言

​	本篇主要是对 RocketMQ的集群搭建的几种方式做简单介绍及其优缺点，便于自己在项目中，根据自己的业务需要做技术选型。在上一篇中，我们介绍了RocketMQ的物理部署结构，知道了它是由NameServer、Producer、Consumer和Broker来组成，各自的作用也做了说明。而这篇介绍的集群的搭建，这里的“集群”指的就是 Broker集群的搭建。

​	Broker 分为 Master 和 Slave，一个 Master 可以对应多个 Slave，但是一个 Slave 只能对应一个 Master，Master 和 Slave 的对应关系通过指定相同的 BrokerName，不同的 BrokerId 来定义，BrokerId为 0 表示 Master，非 0 表示 Slave。Master 也可以部署多个。

​	推荐的几种 Broker 集群部署方式，这里的 Slave 不可写，但可读，类似于 Mysql 主备方式。



# 1.单 Master

> ​	即系统所有的消息都存储在一个 Broker上，这种方式风险较大，一旦 Broker 重启或者宕机时，会导致整个服务不可用。这种只适合入门helloworld实例，强烈不建议线上环境使用。



# 2.多 Master 模式

> ​	一个集群无Slave，全是Master，例如2个Master或者3个Master。

**优点**

> ​	配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。

**缺点**

> ​	单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。

这里介绍下【异步刷盘】、【同步刷盘】。

【异步刷盘】：ASYNC_FLUSH， 



# 3.多 Master 多 Slave 模式，异步复制



# 4.多 Master 多 Slave 模式，同步双写











