---
title:  RocketMQ系列13--Consumer消费端-Push
date:  2018-09-03
categories:  RocketMQ 
tags: [rocketmq,Producer,Consumer,消费] 
	 
---

# 0.概述 #

任何消息中间件,消费者端一般有2种方式从消息中间件获取消息并消费:

**Push方式:**由消息中间件(MQ消息服务器代理)主动将消息推送给消费者。采用Push方式，可以尽可能的实时将消息发送给消费者进行消费。但是，当你的消费者消费消息的能力比较弱的时候，但是MQ仍然会不断的向消费者push最新的消息，消费者端的缓冲区可能会溢出，导致异常。

Pull方式: 由消费者客户端主动向消息中间件（MQ消息服务器代理）拉取消息，采用pull方式，需要关注的时消费者端去拉取的频率。如果每次pull的时间间隔比较久，会增加消息的延迟，即消息到达消费者的时间加长，MQ的消息的会堆积。若每次的pull时间间隔短，有可能在MQ中这段时间内正好没有消息可以消费，那么会产生很多无效的RPC开销，影响整个MQ整体的网络性能。




# 1.RocketMQ消息订阅 #

同样，RocketMQ消息订阅有两种模式，一种是Push模式，即MQServer主动向消费端推送；另外一种是Pull模式，即消费端在需要时，主动到MQServer拉取。通过研究源码可知，RocketMQ的消费方式都是基于拉模式拉取消息的。
那么在RocketMQ中如何解决以做到高效的消息消费呢？

答案就是在rocketmq中有一种长轮询机制（对普通轮询的一种优化），来平衡上面push/pull模型的各自缺点。基本设计思路：

消费者端如果第一次尝试pull消息失败（比如：broker端没有消息可以被消费），这是并不立即给消费者端返回response的响应，而是先hold住并挂起该请求（将该请求保存至pullRequestTable本地缓存变量中），然后broker端的后台独立线程（PullRequestHoldService）会从pullRequestTable本地缓存变量中不断去取信息。	

# 2.demo代码 #


# 3.RocketMQ消息消费的长轮询机制 #



# 4.Push方式的启动流程分析 #



# 5.Push消费模式流程简析 #


# 6.参考 #















