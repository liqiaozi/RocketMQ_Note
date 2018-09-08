---
title:  RocketMQ系列15--消息模式
date:  2018-09-08
categories:  RocketMQ 
tags: [rocketmq,消息模式] 
	 
---

# 0.介绍 #

在RocketMQ中,它没有遵循JMS的规范,而是有一套自定义的机制,简单来说,都是使用订阅主题的方式取发送和接受消息的,但是支持2种消息模式:集群和广播.

```
/**
 * 消息模式
 */
public enum MessageModel {
    /**
     * 广播模式
     */
    BROADCASTING("BROADCASTING"),
    /**
     * 集群模式
     */
    CLUSTERING("CLUSTERING");

    private String modeCN;

    MessageModel(String modeCN) {
        this.modeCN = modeCN;
    }

    public String getModeCN() {
        return modeCN;
    }
}
```

> 集群模式: 设置 消费端对象属性(messageModel)为:MessageModel.CLUSTERING,这种方式可以达到类似于ActiveMQ水平扩展负载均衡消费消息的实现,比较特殊的是,这种方式可以支持先发送数据(也就是producer端先发送数据到MQ),消费端订阅主题发生在生产端之后,也可以接收数据。




# 1.集群模式 #



对于RocketMQ而言，无论是producer还是consumer端，都维护者一个groupname的属性，我们在之前也介绍过，通过ConsumerGroup的机制，消费端实现了天然的负载均衡。也就是说，MQ会将消息平均分发到该消费者组的每个consumer上，这样就要意味着我们可以很方便通过加机器的方式来实现水平扩展！

默认情况下，RocketMQ设置为集群消息模式。至于消息的分发策略，是可以设置策略的：
消息分发策略接口：AllocateMessageQueueStrategy。
默认有以下几种分发策略，要使用哪种策略，只需要示例话对应的对象即可。
```
AllocateMachineRoomNearby()
AllocateMessageQueueAveragely()
AllocateMessageQueueAveragelyByCircle()
AllocateMessageQueueByConfig()
AllocateMessageQueueByMachineRoom()
AllocateMessageQueueConsistentHash()
```



# 2.广播模式 #

 广播消息：设置消费端对象属性为：MessageModel.BROADCASTING,这种模式就是相当于生产端发送数据到MQ，多个消费端都可以获取到数据。

```
consumer.setMessageModel(MessageModel.BROADCASTING);
```

这样，假设producer发送了10条消息，则订阅了该topic的2个消费者，都会接收到这10条消息进行消费。而使用集群模式，会可能会出现consumer1消费序号1-6的消息，而consumer2消费序号7-10的消息。


