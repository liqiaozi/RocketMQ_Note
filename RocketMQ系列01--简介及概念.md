---
title:  RocketMQ系列01--简介及概念
date:  2018-08-27 
categories:  RocketMQ 
tags: rocketmq ; 概念
---


# 1.RocketMQ是什么？

![](http://img3.tbcdn.cn/5476e8b07b923/TB1rdyvPXXXXXcBapXXXXXXXXXX)

RocketMQ：

- 是一个队列模型的消息中间件，具有高性能、高可靠、高实时、分布式的特点；

- Producer、Consumer、队列都可以分布式部署；

- Producer向一些队列轮流发送消息，这些队列集合称为"Topic"。Consumer如果做广播消费，则一个Consumer实例消费这个Topic对应的所有的队列，如果是集群消费，则多个Consumer实例平均消费这个topic队列集合；

- 保证严格的消息顺序；

- 提供丰富的消息拉去模式；

- 高效的订阅者水平扩展能力；

- 实时的消息订阅机制；

- 亿级消息堆积能力。

  RocketMQ是Java语言编写，基于通信框架Netty。



# 2.RocketMQ物理部署结构

![](http://img3.tbcdn.cn/5476e8b07b923/TB18GKUPXXXXXXRXFXXXXXXXXXX)



从上图中，我们可以看到消息队列的主要角色有：

- NameServer:无状态节点，供 Producer 和 Consumer 获取Broker地址。
- Broker:MQ的服务器，消息的中转角色，负责存储和转发消息。
- Producer:发送消息到消息队列(消息生产者)。
- Consumer:从消息队列接收消息（消息消费者）。

RocketMQ网络部署特点：

- 　　NameServer是一个无状态的节点，可集群部署，节点之间无任何信息同步。

- 　　Broker:分为Master和Slave.一个Master可以有多个Slave,但是一个Slave只能对应一个Master.Master和Slave的对应关系是通过指定 “相同的BrokerName,不同的BrokerId"来定义。BrokerId=0表示Master,非0表示Slave。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有的NameServer。

- 　　Producer与NamerServer中的任意一个节点建立长连接，定期从NameServer中取Topic的路由信息，并与提供Topic服务的Master建立长连接，定时向Master发送心跳。可集群部署。

- 　　Consumer与NamerServer中的任意一个节点建立长连接，定期从NameServer中取Topic的路由信息，并与提供Topic服务的Master和Slave建立长连接，且定时向Master和Slave发送心跳。Consumer既可以从 Master 订阅消息，也可以从 Slave 订阅消息，订阅规则由 Broker 配置决定。





# 3.RocketMQ逻辑部署结构

![](http://img3.tbcdn.cn/5476e8b07b923/TB1lEPePXXXXXX8XXXXXXXXXXXX)



**Producer Group**

> ​　　用来表示一个収送消息应用，一个 Producer Group 下包含多个 Producer 实例，可以是多台机器，也可以是一台机器的多个迕程，戒者一个迕程的多个 Producer 对象。一个 Producer Group 可以収送多个 Topic消息。作用有：
>
> ​	1.标识一类producer;
>
> ​	2.查询返个収送消息应用下有多个 Producer 实例;
>
> ​	3.収送分布式事务消息时，如果 Producer 中途意外宕机，Broker 会主劢回调 Producer Group 内的任意一台机器来确讣事务状态。

**Consumer Group**

> ​　　用来表示一个消费消息应用，一个 Consumer Group 下包含多个 Consumer 实例，可以是多台机器，也可以是多个迕程，戒者是一个迕程的多个 Consumer 对象。一个 Consumer Group 下的多个 Consumer 以均摊方式消费消息，如果设置为广播方式，那举返个 Consumer Group 下的每个实例都消费全量数据。

- Topic：消息的逻辑管理单位；

- Message Queue:消息物理管理单位，一个Topic可以有若干个Queue;

- Message:消息

  - body：消息体，用于携带消息的内容。
  - key:消息key,用于区分不同的消息，一般是业务id信息，根据key可查询到消息；
  - tag:消息tag,用于不同的订阅者来过滤消息。

  ​

# **4**.概念术语

- 广播消费

>　　 一条消息被多个 Consumer 消费，即使返些 Consumer 属亍同一个 Consumer Group，消息也会被 Consumer Group 中的每个 Consumer 都消费一次，广播消费中的 Consumer Group 概念可以讣为在消息划分方面无意义。

- 集群消费

> ​　　一个 Consumer Group 中的 Consumer 实例平均分摊消费消息。例如某个 Topic 有 9 条消息，其中一个Consumer Group 有 3 个实例（可能是 3 个迕程，戒者 3 台机器），那举每个实例只消费其中的 3 条消息。在 CORBA Notification 规范中，无此消费方式。
> ​	在 JMS 规范中，JMS point-to-point model 类似，但是 RocketMQ 的集群消费功能大等于 PTP 模型。
> ​	因为 RocketMQ 单个 Consumer Group 内的消费者类似于 PTP，但是一个 Topic/Queue 可以被多个 Consumer Group 消费。

- 顺序消息

> 　　消费消息的顺序要同发送消息的顺序一致，在 RocketMQ 中，主要指的是局部顺序，即一类消息为满足顺序性，必须 Producer 单线程顺序发送，且发送到同一个队列，这样 Consumer 就可以按照 Producer 发送的顺序去消费消息。

- 普通顺序消息

> ​	顺序消息的一种，正常情况下可以保证完全的顺序消息，但是一旦发生通信异常，Broker 重启，由于队列总数发生发化，哈希取模后定位的队列会发化，产生短暂的消息顺序不一致。
> 如果业务能容忍在集群异常情况（如某个 Broker 宕机或者重启）下，消息短暂的乱序，使用普通顺序方式比较合适。

- 严格的顺序消息

> ​　　顺序消息的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式 Failover 特性，即 Broker 集群中只要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。

> 　　如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟的服务不可用。（依赖同步双写，主备自动切换，自动切换功能目前还未实现）
> 目前已知的应用只有数据库 binlog 同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序，推荐使用普通的顺序消息。



# 5.参考

1.RocketMQ 开发指南（V3.2.4）

2.[RocketMQ——初识RocketMQ](https://blog.csdn.net/gwd1154978352/article/details/80654314)













