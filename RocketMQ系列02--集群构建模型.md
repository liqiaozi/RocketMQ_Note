---
title:  RocketMQ系列02--集群构建模型
date:  2018-08-27 
categories:  RocketMQ 
tags: rocketmq ; 集群 
---


# 0.前言


　　本篇主要是对 RocketMQ的集群搭建的几种方式做简单介绍及其优缺点，便于自己在项目中，根据自己的业务需要做技术选型。在上一篇中，我们介绍了RocketMQ的物理部署结构，知道了它是由NameServer、Producer、Consumer和Broker来组成，各自的作用也做了说明。而这篇介绍的集群的搭建，这里的“集群”指的就是 Broker集群的搭建。



　　Broker 分为 Master 和 Slave，一个 Master 可以对应多个 Slave，但是一个 Slave 只能对应一个 Master，Master 和 Slave 的对应关系通过指定相同的 BrokerName，不同的 BrokerId 来定义，BrokerId为 0 表示 Master，非 0 表示 Slave。Master 也可以部署多个。

​　　推荐的几种 Broker 集群部署方式，这里的 Slave 不可写，但可读，类似于 Mysql 主备方式。



# 1.单 Master

> 　　即系统所有的消息都存储在一个 Broker上，这种方式风险较大，一旦 Broker 重启或者宕机时，会导致整个服务不可用。这种只适合入门helloworld实例，强烈不建议线上环境使用。



# 2.多 Master 模式

> ​	一个集群无Slave，全是Master，例如2个Master或者3个Master。

**优点**

> ​　　配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。

**缺点**

> ​	单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。

　　这里介绍下【异步刷盘】、【同步刷盘】、【Broker间数据同步/复制】。

【异步刷盘】：ASYNC_FLUSH,生产者发送的每一条消息并不是立即保存到磁盘,而是暂时缓存起来,然后就返回生产者成功。然后再异步的将缓存数据保存到磁盘，这里有2种情况：

	1.定期将缓存种更新的数据进行刷盘；
	2.当缓存中更新的数据条数达到某一设定值进行刷盘。

这种方式会存在消息丢失（场景：在还未来得及同步到磁盘的时候宕机），但是性能很好，rokectmq默认是这种模式。

【同步刷盘】：生产者发送的每一条消息都在保存到磁盘成功后才返回告诉生产者成功。这种方式不会存在消息丢失的问题，但是有很大的磁盘IO开销，性能有一定影响。

【Broker Replication】:Broker间数据同步/复制。集群环境下需要部署多个Broker，Broker分2种角色：Master和Slave。
	
	1.一种是master,Master即可以读，也可以写，设置brokerId=0,只能设置一个。
	2.另外一种是slave,只允许读，其brokerId为非0。

　　一个master与多个slave通过指定相同的brokerName被归为一个 broker set（broker集群）。通常生产环境中，我们至少需要2个broker Set。Broker Replication指的就是slave 获取或者复制master的数据。
	
	1.sync Broker: 生产者发送的每一条消息都至少同步复制到一个slave后才返回告诉生产者成功，即"同步双写"；
	2.Async Broker: 生产者发送的每一条消息只要写入master就返回告诉生产者成功，然后再"异步复制"到slave.
	
# 3.多 Master 多 Slave 模式，异步复制

> 每个Master配置一个或多个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟，毫秒级。

**优点**

> 　　即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为Master宕机后，消费者仍然可以从Slave消费，此过程对应用透明。不需要人工干预。性能同多Master模式几乎一样。

**缺点**

> 　　Master宕机，磁盘损坏情况，会丢失少量消息。当producer向master发送完数据时，这时候还没等到master向slave做同步，master宕机了，则在master上的数据会丢失，消费者在salve上消费不到刚刚的消息。


# 4.多 Master 多 Slave 模式，同步双写

> 每个Master配置一个或多个Slave，有多对Master-Slave，HA采用同步双写方式，主备都写成功，向应用返回成功。

**优点**

> 数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高。

**缺点**

>  　　性能比异步复制模式略低，大约低10%左右，发送单个消息的RT会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。


在下篇中，我们会进行 2-master集群环境的搭建和配置。

# 参考 #

1.《RocketMQ用户指南》












