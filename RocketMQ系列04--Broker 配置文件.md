---
title:  RocketMQ系列04--Broker 配置文件
date:  2018-08-28
categories:  RocketMQ 
tags: [rocketmq,Broker,配置文件] 
	 
---

# Broker配置 #

```
#所属集群名字
#注意:一个集群中如果有多个master，那么每个master配置的 brokerClusterName 名字应该一样，要不然识别不了对方，不知道是一个集群内部的
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
#建议:按配置文件文件名来匹配
brokerName=broker-a|broker-b
#0 表示 Master， >0 表示 Slave
brokerId=0
#nameServer地址，分号分割
#此处nameserver跟host配置相匹配，9876为默认rk服务默认端口
#broker启动时会跟nameserver建一个长连接，broker通过长连接才会向nameserver发新建的topic主题，然后java的客户端才能跟nameserver端发起长连接，向nameserver索取topic，找到topic主题之后，判断其所属的broker，建立长连接进行通讯，这是一个至关重要的路由的概念，重点，也是区别于其它版本的一个重要特性
namesrvAddr=rocketmq-nameserv1:9876;rocketmq-nameserv2:9876

#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911

#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88

#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
#消息实际存储位置，和ConsumeQueue是mq的核心存储概念，之前搭建2m环境的时候创建在store下面，用于数据存储，consumequeue是一个逻辑的概念，消息过来之后，consumequeue并不是把消息所有保存起来，而是记录一个数据的位置，记录好之后再把消息存到commitlog文件里
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort

#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000

#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER

#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH

#checkTransactionMessageEnable=false

#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128

```











