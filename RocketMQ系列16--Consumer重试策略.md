---
title:  RocketMQ系列16--Consumer重试策略
date:  2018-09-09
categories:  RocketMQ 
tags: [rocketmq,重试] 
	 
---

# 1.消息去重简介 #
RocketMQ提供了消息重试机制,这时一些其他消息队列没有的功能.我们可以依靠这个优秀的机制,而不用在开发种增加更多的业务代码去实现。

Consumer消费消息失败后，要提供一种重试机制，令消息再消费一次。Consumer消费消息失败通常可以认为有以下几种情况：

- 1.由于消息本身的原因，例如反序列化失败，消息数据本身无法处理（例如充话费，当前手机号被注销，无法充值）等；

这种错误通常需要跳过这条消息，再消费其他消息，而这条失败的消息即使立刻重试消费，99%也不成功，所以最好提供一种定时重试策略，即过10s后在重试。

- 2.由于依赖的下游应用服务不可用，例如db链接不可用，外网不可达等。

遇到这种错误，即使跳过当前的失败的消息，消费其他消息同样也会报错，这种情况建议应用sleep 30s,再消费吓一跳消息，这样可以减轻Broker重试消息的压力。

# 2.消息重试分类 #

对于MQ，可能存在各种异常和情况，导致消息无法最终被consumer消费，因此就有了消息失败重试机制。很显然，消息重试分为2种：Producer端重试和Consumer端重试。



## 2.1 producer端 ##


生产端的消息失败，也就是Producer往MQ上发消息没有发送成功，比如网络抖动导致生产者发送消息到MQ失败。这种消息失败重试我们可以手动设置发送失败重试次数，代码如下：
```
 	//发送失败重试次数
 	producer.setRetryTimesWhenSendFailed(3);
 	...
	// 3s内没有发送成功,即重试发送.
    SendResult sendResult = producer.send(msg,3000);
```

send（）方法内部重试逻辑：
1. 至多重试设置的重试的次数，默认为3次；
2. 如果发送失败，则轮转到下一个broker;
3. 这个方法的总耗时事件不超过sendMsgTimeout设置的值，默认为10s。

所以，如果本身向broker发送消息产生超时异常，就不会再做重试。

以上策略仍然不能保证消息一定发送成功，为保证消息一定成功，建议这样做：
如果调用send同步方法发送消息失败，则尝试将消息存储到db中，有后台线程定时重试，保证消息一定到达broker.



## 2.2 consumer端 ##
消费端消费失败的情况分为2种：exception 和 timeout。

### exception ###

消息正常到达了消费端，结果消费者发生了异常，处理失败了。（发序列化失败，消息本身无法处理等）。

我们可以带着问题去找答案：

> 消费者消费消息的状态有哪些？
> 如果消费失败，mq采用什么策略进行重试？
> 假设10条试拒中，某一条消息消费失败，是消息重试这10条消息呢？还是只是重试失败的某一条？
> 再重试的过程中，需要保证不重复消费吗？

消息消费的状态：

```
public enum ConsumeConcurrentlyStatus {
    /**
     * 消费成功.
     */
    CONSUME_SUCCESS,
    /**
     * 消费失败,稍后重试消费.
     */
    RECONSUME_LATER;
}
```
查看broker的启动日志：

```
2018-08-28 22:14:12 INFO main - messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
2018-08-28 22:14:12 INFO main - flushDelayOffsetInterval=10000
2018-08-28 22:14:12 INFO main - cleanFileForciblyEnable=true
2018-08-28 22:14:28 INFO main - user specfied name server address: rocketmq-nameserv1:9876;rocketmq-nameserv2:9876

```
可以看到消息失败重试的策略：
```
2018-08-28 22:14:12 INFO main - messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```
直到2个小时候消费还是失败，那么这条消息就会终止发送给消费者了。再实际中，我们并不会允许这么多次消费的，这样做很浪费资源，我们会把这条消息存储到db中，手动消费。

```
	catch (UnsupportedEncodingException e) {
    e.printStackTrace();
    if (msgs.get(0).getReconsumeTimes() == 3) {
        // 该条消息可以存储到DB或者LOG日志中，或其他处理方式
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;// 成功
    } else {
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;// 重试
    }
}
```

### timeout ###

由于网络原因导致消息压根就没从MQ到消费者上，那么RocketMQ内部会不断的尝试发送这条消息，直至发送成功为止.(集群中的一个broker失败，就尝试另外一个broker)。延续exception的思路，也就是消费者端没有返回消息的消费状态，无论是消费成功还是稍后重试。这样producer端就认为消息还是没有到达消费者端。

场景模拟：
(1) 同在一个消费者组的2个消费者：consumer1和consumer2;
(2) consumer1的业务代码中暂停1分钟，并且不发送状态给MQ。

```
consumer.registerMessageListener(new MessageListenerConcurrently() {
@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
    try {                   
            String topic = msg.getTopic();
            String msgBody = new String(msg.getBody(),"utf-8");
            String tags = msg.getTags();
            System.out.println("收到消息：" + " topic：" + topic + " ,tags：" + tags + " ,msg：" + msgBody);

            // 表示业务处理时间
            System.out.println("=========开始暂停==========");
            Thread.sleep(60000);
        }
    } catch (Exception e) {
        e.printStackTrace();
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;// 重试
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
}
});
```
(3)先启动consumer1，再启动producer,发送一条消息。
(4)发现consumer1接收到了消息，但是因为consumer阻塞，没有返回消息消费的状态；
(5)紧接着，启动consumer2，发现consumer接收到了消息，并且消费了消息返回了消费状态。










