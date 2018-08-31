---
title:  RocketMQ系列11--Producer 顺序消息
date:  2018-08-31
categories:  RocketMQ 
tags: [rocketmq,Producer,顺序消息] 
	 
---

# 0.前言 #

顺序消息（FIFO消息）是MQ提供的一种严格按照顺序进行发布和消费的消息类型。顺序消息指消息发布和消息消费都按照顺序进行。

- 顺序发布: 对于指定的一个Topic,客户端将按照一定的先后顺序发送消息；
- 顺序消费: 对于指定的一个Topic,按照一定的先后顺序接收消息，即先发送的消息一定会被消费段先接受到并处理。

　　例如：一笔订单产生了3条消息，分别是：订单创建、订单付款、订单完成。消费时，要按照顺序一次消费才有意义。于此同时多笔订单之间又是可以并行消费的。

**全局顺序**

对于指定的一个Topic,所有的消息按照严格的先入先出（FIFO）的顺序进行发布和消费。

![](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/49319/cn_zh/1534917028902/%E5%85%A8%E5%B1%80%E9%A1%BA%E5%BA%8F.png)


请参考 阿里的《[顺序消息](https://help.aliyun.com/document_detail/49319.html?spm=a2c4g.11186623.4.6.6a0a292dCAyla4)》说明。

# 1.RocketMQ的顺序消息的实现 #

　　顺序消息主要是指局部顺序，即生产者通过将某一类消息发送至同一个队列来实现。与发生普通消息相比，在发送顺序消息时要对同一类型的消息选择同一个队列，即同一个　**MessageQueue**　对象。 目前RocketMQ定义了选择MessageQueue对象的接口MessageQueueSelector，里面有方法select(final List mqs, final Message msg, final Object arg)，并且RocketMQ默认实现了提供了两个实现类SelectMessageQueueByHash和SelectMessageQueueByRandoom，即根据arg参数通过Hash或者随机方式选择MessageQueue对象。 为了业务层根据业务需要能自定义选择规则，也可以在业务层自定义选择规则，然后调用DefaultMQProducer.send(Message msg, MessageQueueSelector selector, Object arg)方法完成顺序消息的方式。 

　　消息发布是有序的含义：producer发送消息应该是依次发送的，所以要求发送消息的时候保证：

- 消息不能异步发送，同步发送的时候才能保证broker收到是有序的。
- 每次发送选择的是同一个MessageQueue


# 2.Producer实现 #

　　参考网上的例子，编写如下的producer发送顺序消息。
```
public class Producer {

    public static void main(String[] args) throws UnsupportedEncodingException, RemotingException, MQClientException, MQBrokerException {
        try {
            //设置 producerGroup 名称
            DefaultMQProducer producer = new DefaultMQProducer("Order_Producer_Group");
            //设置 nameserver地址.
            producer.setNamesrvAddr("192.168.81.132:9876;192.168.81.134:9876");
            producer.start();

            // 订单创建 订单支付 订单完成
            String[] tags = new String[] {"TagA", "TagB", "TagC"};

            List<Order> orders = buildOrders();
            for(int i = 0; i < orders.size(); i++){
                String body = "Hello Rocket" + orders.get(i);
                long orderId = orders.get(i).getOrderId();

                Message message = new Message("Topic_Order",
                        tags[i % tags.length],
                        "KEY" + i,
                        body.getBytes());

                SendResult sendResult = producer.send(message, new SelectMessageQueueByHash(), orderId);
                System.out.println("content=" + body + ". sendResult = " + sendResult);
            }

            producer.shutdown();
        } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {
            e.printStackTrace();
        }
    } // end main.

    /**
     * 生成模拟订单数据
     */
    private static List<Order> buildOrders() {
        List<Order> orderList = new ArrayList<Order>();

        Order orderDemo = new Order();
        orderDemo.setOrderId(15103111039L);
        orderDemo.setDesc("创建");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103111065L);
        orderDemo.setDesc("创建");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103111039L);
        orderDemo.setDesc("付款");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103117235L);
        orderDemo.setDesc("创建");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103111065L);
        orderDemo.setDesc("付款");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103117235L);
        orderDemo.setDesc("付款");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103111065L);
        orderDemo.setDesc("完成");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103111039L);
        orderDemo.setDesc("推送");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103117235L);
        orderDemo.setDesc("完成");
        orderList.add(orderDemo);

        orderDemo = new Order();
        orderDemo.setOrderId(15103111039L);
        orderDemo.setDesc("完成");
        orderList.add(orderDemo);

        return orderList;
    }


    }

@Data
class Order{
    private long orderId;

    private String desc;

}

```



# 3.Consumer实现 #

```
public class Consumer {

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("Order_Consumer_Group");
        consumer.setNamesrvAddr("192.168.81.132:9876;192.168.81.134:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("Topic_Order", "*");

        consumer.registerMessageListener(new MessageListenerOrderly() {
            Random random = new Random();
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                context.setAutoCommit(true);
                System.out.print(Thread.currentThread().getName() + " Receive New Messages: " );
                for (MessageExt msg: msgs) {
                    System.out.println(msg + ", content:" + new String(msg.getBody()));
                }
                try {
                    //模拟业务逻辑处理中...
                    TimeUnit.SECONDS.sleep(random.nextInt(10));
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return ConsumeOrderlyStatus.SUCCESS;
            }

        });

        consumer.start();
        System.out.printf("Consumer Started.%n");
    }

}

```


# 4.结果 #

**producer端发送的消息**：

```
content=Hello RocketOrder(orderId=15103111039, desc=创建). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002A62E, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=0], queueOffset=0]
content=Hello RocketOrder(orderId=15103111065, desc=创建). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002A6E5, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=2], queueOffset=0]
content=Hello RocketOrder(orderId=15103111039, desc=付款). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002A79C, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=0], queueOffset=1]
content=Hello RocketOrder(orderId=15103117235, desc=创建). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518400002A9F0000000000033BBC, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-a, queueId=0], queueOffset=0]
content=Hello RocketOrder(orderId=15103111065, desc=付款). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002A853, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=2], queueOffset=1]
content=Hello RocketOrder(orderId=15103117235, desc=付款). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518400002A9F0000000000033C73, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-a, queueId=0], queueOffset=1]
content=Hello RocketOrder(orderId=15103111065, desc=完成). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002A90A, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=2], queueOffset=2]
content=Hello RocketOrder(orderId=15103111039, desc=推送). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002A9C1, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=0], queueOffset=2]
content=Hello RocketOrder(orderId=15103117235, desc=完成). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518400002A9F0000000000033D2A, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-a, queueId=0], queueOffset=2]
content=Hello RocketOrder(orderId=15103111039, desc=完成). sendResult = SendResult [sendStatus=SEND_OK, msgId=C0A8518600002A9F000000000002AA78, messageQueue=MessageQueue [topic=Topic_Order, brokerName=broker-b, queueId=0], queueOffset=3]

```

**consumer端消费消息**

```
ConsumeMessageThread_4 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=0, sysFlag=0, bornTimestamp=1535694773214, bornHost=/192.168.81.1:61621, storeTimestamp=1535723526505, storeHost=/192.168.81.132:10911, msgId=C0A8518400002A9F0000000000033BBC, commitLogOffset=211900, bodyCRC=1826716009, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, KEYS=KEY3, WAIT=true, TAGS=TagA}, body=51]], content:Hello RocketOrder(orderId=15103117235, desc=创建)
ConsumeMessageThread_5 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=0, sysFlag=0, bornTimestamp=1535694772791, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553085, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002A62E, commitLogOffset=173614, bodyCRC=1934949583, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=4, KEYS=KEY0, WAIT=true, TAGS=TagA}, body=51]], content:Hello RocketOrder(orderId=15103111039, desc=创建)
ConsumeMessageThread_6 Receive New Messages: MessageExt [queueId=2, storeSize=183, queueOffset=0, sysFlag=0, bornTimestamp=1535694773081, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553244, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002A6E5, commitLogOffset=173797, bodyCRC=1291356221, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, KEYS=KEY1, WAIT=true, TAGS=TagB}, body=51]], content:Hello RocketOrder(orderId=15103111065, desc=创建)
ConsumeMessageThread_6 Receive New Messages: MessageExt [queueId=2, storeSize=183, queueOffset=1, sysFlag=0, bornTimestamp=1535694773360, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553455, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002A853, commitLogOffset=174163, bodyCRC=1081168709, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, KEYS=KEY4, WAIT=true, TAGS=TagB}, body=51]], content:Hello RocketOrder(orderId=15103111065, desc=付款)
ConsumeMessageThread_4 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=1, sysFlag=0, bornTimestamp=1535694773369, bornHost=/192.168.81.1:61621, storeTimestamp=1535723526572, storeHost=/192.168.81.132:10911, msgId=C0A8518400002A9F0000000000033C73, commitLogOffset=212083, bodyCRC=1617469969, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, KEYS=KEY5, WAIT=true, TAGS=TagC}, body=51]], content:Hello RocketOrder(orderId=15103117235, desc=付款)
ConsumeMessageThread_5 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=1, sysFlag=0, bornTimestamp=1535694773157, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553299, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002A79C, commitLogOffset=173980, bodyCRC=2145200055, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=4, KEYS=KEY2, WAIT=true, TAGS=TagC}, body=51]], content:Hello RocketOrder(orderId=15103111039, desc=付款)
ConsumeMessageThread_6 Receive New Messages: MessageExt [queueId=2, storeSize=183, queueOffset=2, sysFlag=0, bornTimestamp=1535694773377, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553489, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002A90A, commitLogOffset=174346, bodyCRC=339615115, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, KEYS=KEY6, WAIT=true, TAGS=TagA}, body=51]], content:Hello RocketOrder(orderId=15103111065, desc=完成)
ConsumeMessageThread_5 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=2, sysFlag=0, bornTimestamp=1535694773404, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553494, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002A9C1, commitLogOffset=174529, bodyCRC=742301160, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=4, KEYS=KEY7, WAIT=true, TAGS=TagB}, body=51]], content:Hello RocketOrder(orderId=15103111039, desc=推送)
ConsumeMessageThread_4 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=2, sysFlag=0, bornTimestamp=1535694773411, bornHost=/192.168.81.1:61621, storeTimestamp=1535723526623, storeHost=/192.168.81.132:10911, msgId=C0A8518400002A9F0000000000033D2A, commitLogOffset=212266, bodyCRC=875031775, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, KEYS=KEY8, WAIT=true, TAGS=TagC}, body=51]], content:Hello RocketOrder(orderId=15103117235, desc=完成)
ConsumeMessageThread_5 Receive New Messages: MessageExt [queueId=0, storeSize=183, queueOffset=3, sysFlag=0, bornTimestamp=1535694773426, bornHost=/192.168.81.1:61618, storeTimestamp=1535723553517, storeHost=/192.168.81.134:10911, msgId=C0A8518600002A9F000000000002AA78, commitLogOffset=174712, bodyCRC=731015545, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=Topic_Order, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=4, KEYS=KEY9, WAIT=true, TAGS=TagA}, body=51]], content:Hello RocketOrder(orderId=15103111039, desc=完成)

```



# 5.Producer发送顺序消息过程 #

　　上面我们已经说过，要保证消息是顺序消息，在发送顺序消息时要对同一类型的消息选择同一个队列，即同一个　**MessageQueue**　对象。
　　在这里，我们要保证的是同一订单的消息要投放到同一个队列中，所以我们根据订单号去hash运算，一样的订单号得到的hash值可定是相同的，只要保证其队列的个数是不变的，则同一订单号的消息会被放到同一个队列中。

之前发送普通消息时，我们调用：
```
public SendResult send(Message msg)
```
但发送顺序消息时，由于我们要把同一特征的消息放到同一队列中，所以我们使用：

```
public SendResult send(Message msg, MessageQueueSelector selector, Object arg)
```
根据传入的arg值，选择路由到同一个队列中。

## 5.1MessageQueueSelector ##

第二个参数 MessageQueueSelector 是一个接口，rocketmq默认提供了3种实现。
```
public interface MessageQueueSelector {
    MessageQueue select(List<MessageQueue> var1, Message var2, Object var3);
}
```
![](https://upload-images.jianshu.io/upload_images/11560519-b735612943499713.png)

```
/**
 *  发送消息，随机选择队列
 */
public class SelectMessageQueueByRandom implements MessageQueueSelector {
    private Random random = new Random(System.currentTimeMillis());

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = random.nextInt(mqs.size());
        return mqs.get(value);
    }
}
```
> 随机选择，也就是谁也不知道它到底会选择谁，这种效率其实很差，没有负载均衡，谁也不知道会不会堵塞起来，谁也不知道某个队列是否已经塞满

```
/**
 * 使用哈希算法来选择队列，顺序消息推荐该实现.
 */
public class SelectMessageQueueByHash implements MessageQueueSelector {

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = arg.hashCode();
        if (value < 0) {
            value = Math.abs(value);
        }

        value = value % mqs.size();
        return mqs.get(value);
    }
}
```
> 我们每个传递进入的对象都会被哈希算法计算出 一个哈希值，比如我们传递的是订单号，那么无疑我们可以保证相同的订单号可以传递给相同的topic去处理，那么只要再保证是一致的tag就可以保证顺序的一致性啦

```
/**
 * 根据机房来选择发往哪个队列，支付宝逻辑机房使用
 */
public class SelectMessageQueueByMachineRoom implements MessageQueueSelector {
    private Set<String> consumeridcs;

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        return null;
    }

    public Set<String> getConsumeridcs() {
        return consumeridcs;
    }

    public void setConsumeridcs(Set<String> consumeridcs) {
        this.consumeridcs = consumeridcs;
    }
}
```
> 机房选择，算法是木有啦，应该是根据ip地址去区分。

　　同理，我们在使用时可以根据自己的业务需要来选择合适的方案，也可以自定义：

```
SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                        Integer id = (Integer) arg;
                        int index = id % mqs.size();
                        return mqs.get(index);
                    }
                }, arg);
```


## 5.2 顺序消息发送 ##

底层调用

```
 private SendResult sendSelectImpl(
        Message msg,
        MessageQueueSelector selector,
        Object arg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback, final long timeout
    )
```

```
 private SendResult sendSelectImpl(
        Message msg,
        MessageQueueSelector selector,
        Object arg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback, final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // 确定producer的状态时Running.
        long beginStartTime = System.currentTimeMillis();
        this.makeSureStateOK();
        // 校验message的格式.
        Validators.checkMessage(msg, this.defaultMQProducer);

        // 找到topic的路由信息,否则抛出异常.
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            
            MessageQueue mq = null;
            try {
                // 路由到消息要发送的消息队列.
                mq = selector.select(topicPublishInfo.getMessageQueueList(), msg, arg);
            } catch (Throwable e) {
                throw new MQClientException("select message queue throwed exception.", e);
            }

            // 判断是否超时.
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeout < costTime) {
                throw new RemotingTooMuchRequestException("sendSelectImpl call timeout");
            }
            // 如果找到了要发送的消息队列,则发送该消息,否则跑相互异常信息.
            if (mq != null) {
                return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, null, timeout - costTime);
            } else {
                throw new MQClientException("select message queue return null.", null);
            }
        }

        throw new MQClientException("No route info for this topic, " + msg.getTopic(), null);
    }

```

```
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
        //  private final ConcurrentMap<String, TopicPublishInfo> topicPublishInfoTable = new ConcurrentHashMap<String, TopicPublishInfo>();
        TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
        
        // 如果 topic的路由信息查询不到,或者该topic的消息队列还未初始化好,则创建该topic路由
        // 并更新到namesrv上.
        if (null == topicPublishInfo || !topicPublishInfo.ok()) {
            this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
        }

        if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
            return topicPublishInfo;
        } else {
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
            return topicPublishInfo;
        }
    }
```

## 5.3 consumer端接收 ##

　　普通消息consumer注册的监听器是： MessageListenerConcurrently：

```
public void registerMessageListener(MessageListenerConcurrently messageListener) {
        this.messageListener = messageListener;
        this.defaultMQPushConsumerImpl.registerMessageListener(messageListener);
    }
```

　　而顺序消息consumer注册的监听器时是 MessageListenerOrderly：

```
public void registerMessageListener(MessageListenerOrderly messageListener) {
        this.messageListener = messageListener;
        this.defaultMQPushConsumerImpl.registerMessageListener(messageListener);
    }
```

```
public enum ConsumeOrderlyStatus {
    /**
     * Success consumption
     */
    SUCCESS,
    /**
     * Rollback consumption(only for binlog consumption)
     */
    @Deprecated
    ROLLBACK,
    /**
     * Commit offset(only for binlog consumption)
     */
    @Deprecated
    COMMIT,
    /**
     * Suspend current queue a moment
     */
    SUSPEND_CURRENT_QUEUE_A_MOMENT;
}
```
　　本地消费的事务控制，


- ConsumeOrderlyStatus.SUCCESS（提交）
- ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT（挂起一会再消费）

　
　　在此之前还有一个变量ConsumeOrderlyContext context的setAutoCommit()是否自动提交。

　　当SUSPEND_CURRENT_QUEUE_A_MOMENT时，autoCommit设置为true或者false没有区别，本质跟消费相反，把消息从msgTreeMapTemp转移回msgTreeMap，等待下次消费。

当SUCCESS时，autoCommit设置为true时比设置为false多做了2个动作，

    # 本质是删除msgTreeMapTemp里的消息，msgTreeMapTemp里的消息在上面消费时从msgTreeMap转移过来的
	consumeRequest.getProcessQueue().commit()
	# 本质是把拉消息的偏移量更新到本地，然后定时更新到broker
    this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);


　　那么少了这2个动作会怎么样呢，随着消息的消费进行，msgTreeMapTemp里的消息堆积越来越多，消费消息的偏移量一直没有更新到broker导致consumer每次重新启动后都要从头开始重复消费。所以 autoCommit建议设置为true.


　　好的，本篇顺序消息主要介绍了 顺序消息的发送、消费、以及与普通消息的不同。下篇将介绍 事务消息的发送。










