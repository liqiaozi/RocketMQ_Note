---
title:  RocketMQ系列06--整体架构概述
date:  2018-08-30
categories:  RocketMQ 
tags: [rocketmq,源码目录,架构] 
	 
---

# 0. 前言 #

　　本篇,我主要根据 rocketmq的源码目录,简单介绍下 rocketmq各模块的功能,及其各自间的相互调用。接着大体上说一下 其中的nameserver，底层通信和数据存储。但里面的东西还需要大家去阅读源码，仔细体会。这里只是做个引子。我在文章的最后推荐了 【Rocketmq源码阅读】的系列博客，对大家阅读源码可能会有帮助。
废话不多说，直入主题。


# 1. 源码目录 #
　　我这里是下载的 最新的 RocekyMQ 4.4.0-SNAPSHOT版本的[源码](https://github.com/apache/rocketmq/) 
导入到 IDEA中，目录结构如图：



![](https://upload-images.jianshu.io/upload_images/11560519-53e6a18b34657498.png)

- distribution
> 　　存放系统中的脚本，以及一些broker集群的配置文件，用户可以根据自己的项目信息更改配置文件或脚本中的内容，来启动 nameserver 和 broker,集合 example中的Producer/Consumer来做一些小测试。

- broker
> 　　消息代理，起到 串联 Producer/Consumer 和 Store的作用。我们所谓的消息的存储、接收、拉去、推送等操作都是在broker上进行的。

- client
> 　　包含 Producer 和 Consumer，负责消息的发和收。

- common
> 　　通用的常量枚举、基类方法或数据结构，按描述的目标来分包，

- example
> 　　使用样例，包含各种使用方法，Pull/Push模式，广播模式、有序消息，事务消息。

- filter
> 　　过滤器，用于服务端的过滤方式，实现了真正意义的高内聚低耦合的设计思想。
> 　　在使用filter模块的时候需要启动filter的服务。

- logappender logging
> 　　日志相关。

- namesrv
> 　　注册中心，每个broker都会在这里注册，client也会从这里获取broker 的相关信息。

- openmessaging
>

- remoting
> 　　基于Netty实现的额昂罗通信模块，包括server和client,client broker filter 等模块都对他有依赖。

- srvutil
> 　　用来处理命令行的，一个用来配置shutdownHook.目的为了拆分客户端依赖，尽可能减少客户端的依赖。

- store
> 　　负责消息的存储和读取。

- test
> 　　测试用例代码

- tools
> 　　一些工具类，基于他们可以写一些工具来管理、查看MQ系统的一些信息。

# 2. 模块调用图及层次说明 #


![](https://upload-images.jianshu.io/upload_images/11560519-f4bf6ba3b4f5bc93.png)

![](https://upload-images.jianshu.io/upload_images/11560519-aa7a629b10e20a53.png)


# 3. nameserver介绍 #

　　在rocketmq早期版本中，是没有namersrv的，而是用 zookeeper做分布式协调和服务发现的。但是后期阿里根据业务需要自主研发了轻量级的 namesrv，用于注册client服务与broker的请求路由工作，namesrv上不做任何消息的位置存储（之前的zookeeper的位置存储数据会影响整体集群性能）

- 　　rocketmq-namesrv扮演着nameNode角色，记录着运行时消息相关的meta信息以及broker和filtersrv的运行时信息，可以部署集群；
- 　　可以看作是轻量级的zookeeper，但比zookeeper性能更好，可靠性更强；
- 　　rocketmq-namesrv主要是节点之间相互进行心跳检测、数据通信、集群高可靠性，一致性、容错性等方面的核心模块；
- 　　roketmq-namesrv的底层通信机制与Netty进行联系，上层通信与各个模块产生强一致性的对应关系。当broker producer consumer 都运行后，maerserv一共有8类线程：
	- 守护线程
	- 定时任务线程
	- Netty的boss线程
	- NettyEventExecute线程
	- DestroyJavaVM线程
	- Work线程
	- Handler线程
	- RemotingExxecutorThread线程

# 4. 底层通信介绍 #

- ServerHouseKeepingService
> 守护线程，本质是 ChannelEventListner,监听broker的channel变化来更新本地的RouteInfo.

- NSScheduledThread：定时任务线程，定时跑2个线程，第一个：每个10分钟> > 扫描出不活动的broker,然后 从routeInfo中剔除；第二个：每10分钟定时打印configTable的信息。

- NettyBossSelector
> Metty的boss线程，

- NettyEventExecuter
> 一个单独的线程，监听NettyChannel状态变化来通知ChannelEventListener做响应动作。

- DestroyJavaVM
> Java虚拟机解析钩子，已办是当虚拟机关闭时用来清理或者释放资源；

- NettyServerSelector
> Netty的work线程，可能有多个。

- NettyServerWorkerThread_x
> 执行ChannelHandler方法的线程。

- RemotingExecutorThread_x
> 服务端逻辑线程

 




# 6. 推荐 #
[【Rocketmq源码阅读】](https://mp.weixin.qq.com/s/UIPgD7EaaiOArwctdVmJAA)