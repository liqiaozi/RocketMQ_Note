---
title:  RocketMQ系列05--Hello world入门篇
date:  2018-08-29
categories:  RocketMQ 
tags: [rocketmq,hello world,入门,上手] 
	 
---

# 0.前言 #
前面几篇介绍了rocketmq的基本概念,结构,集群搭建方案,上一篇搭建了2m-noslave双主集群模式的broker。接下来，我们下载 rocketmq的源码包，根据里面的示例来进一步了解 rocketmq的各种特性。

源码地址： https://github.com/apache/rocketmq/
我们会参考 example的示例代码来入门。（我这里依然采用的时3.2.6版本）

# 1.pom依赖 #

```
  		<!--rocketmq-->
        <dependency>
            <groupId>com.alibaba.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>3.2.6</version>
        </dependency>
```

# 2.producer示例代码 #

```
@Slf4j
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        //设置 producerGroup 名称,保证不同的业务唯一性.
        DefaultMQProducer producer = new DefaultMQProducer("DingDing");
        //设置 nameserver地址.
        producer.setNamesrvAddr("192.168.81.132:9876;192.168.81.134:9876");
        //启动 producer.
        producer.start();

        // 发送消息.
        for (int i = 0; i < 100; i++) {
            try {

                Message msg = new Message("DingDing_Topic", // topic
                        "sign_up",                           // tags
                        ("员工" + i + "签到").getBytes());        // body

                SendResult sendResult = producer.send(msg);
                log.info("发送结果:" + sendResult);
            }
            catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }

        // 关闭 producer.
        producer.shutdown();
    }
}

```

**步骤**
> 1.设置 producer group name，用来区分不同业务的producer，因此需要保证不同业务改名称不同；
> 
> 2.指定链接的nameserver的地址，集群的话地址间以 ； 分割；
> 
> 3.启动消息生产者；
>
> 4.构造消息对象，发送消息，处理发送结果；
>
> 5.当发送消息结束后，应该关闭producer.

# 3.consumer示例代码 #

```
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("DingDing");
        //设置 nameserver地址.
        consumer.setNamesrvAddr("192.168.81.132:9876;192.168.81.134:9876");
        /**
         * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>
         * 如果非第一次启动，那么按照上次消费的位置继续消费
         */
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("DingDing_Topic", "*");

//        consumer.setConsumeMessageBatchMaxSize(10);

        consumer.registerMessageListener(new MessageListenerConcurrently() {

            // 接收消息.
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                            ConsumeConcurrentlyContext context) {
//                System.out.println(Thread.currentThread().getName() + " 收到消息 " + msgs);
                System.out.println("本批次收到消息的数量 " + msgs.size());
                for(MessageExt msg : msgs){

                    try {
                        String topic = msg.getTopic();
                        String tags = msg.getTags();
                        String msgBody =  new String(msg.getBody(),"utf-8");
                        System.out.println("收到消息--" + " topic:" + topic + " ,tags:" + tags + " ,msg:" +msgBody);
//                        System.out.println();
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
						//	有异常抛出来，不要全捕获了，这样保证不能消费的消息下次重推，每次重新消费间隔：10s,30s,1m,2m,3m
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                    }
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.start();

        System.out.println("Consumer Started.");
    }
}
```

**步骤**
> 1.同producer 需要设置 producer group一样，消费者同样需要设置 consumer group name;
> 2.指定nameserver的地址，需要同要处理的producer的nameserver的地址相同；
> 3.指定消费者要消费的位置（通常设置为：从头开始 CONSUME_FROM_FIRST_OFFSET）；
> 4.订阅要处理的主题 topic 及 细分 tags；
> 5.注册监听器开始监听broker中是否有消息队列，编写消费逻辑；如果没有return success ，consumer会重新消费该消息，直到return success。
> 6.启动消费者实例。



# 4.运行结果 #

**启动顺序**

> 先启动 consumer实例，再启动 producer 实例；默认 是单线程的一条一条的去处理消息，原子性，保证了消息处理的实时性。


**持久化**
RocketMQ是一定会把消息持久化的。

**单批次消息消费的数量**

如果是先启动 consumer，后启动 producer，理论上rocketmq发送的消息的数量都是1。
当然，如果有消息挤压的情况下，设置 consumer.setConsumeMessageBatchMaxSize(10)，可能会出现一次消息的消费数量可能是多条，这里设置的数量是 10，是最大条数是10，而不是每次批量消费的数量都是10.

结果：
1.先启动 Consumer，后启动 Producer，会发现每次的过来的消息数量仍然是1，说明没有消息挤压，实时消费；

2.先启动 Producer,发送一定数量的消息，然后再启动 Consumer,会发现，消费者这边接收到的消息的数量是不固定的。



----------
本次的rocketmq，简单入门就到这里，大家再运行代码的时候，可以看看重要类的属性、方法等，加深了解。
 









