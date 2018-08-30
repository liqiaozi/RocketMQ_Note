---
title:  RocketMQ系列10--Producer介绍
date:  2018-08-31
categories:  RocketMQ 
tags: [rocketmq,Producer,发送消息] 
	 
---
# 0. 前言 #

rocketmq支持3种形式的消息:
- 普通消息
- 顺序消息
- 事务消息

同时支持3种不同的发送方式
- 同步
- 异步
- One-Way

本篇将以 DefaultMQProducer(普通消息)为例讲解 producer的一些属性配置、发送机制和消息结果的处理事项。


# 1. MQProducer属性配置 #

```
     
	# Producer 组名，多个 Producer 如果属于一个应用，发送同样的消息，则应该将它们归为同一组
    private String producerGroup;

    # 在发送消息时，自动创建服务器不存在的topic，需要指定 Key。
	# 建议线下测试时开启，线上环境关闭
    private String createTopicKey = MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC;

    # 在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
    private volatile int defaultTopicQueueNums = 4;

    # 发送消息超时时间，单位毫秒
    private int sendMsgTimeout = 3000;

	# 消息 Body 超过多大开始压缩（Consumer收到消息会自动解压缩），单位字节
    private int compressMsgBodyOverHowmuch = 1024 * 4;

	# 消息同步发送失败最大重试次数，超过这个设置次数后，需要开发者自己去解决。
    private int retryTimesWhenSendFailed = 2;

	# 消息异步发送失败最大重试次数，超过这个设置次数后，需要开发者自己去解决。
    private int retryTimesWhenSendAsyncFailed = 2;

    # 如果发送消息返回 sendResult，但是 sendStatus!=SEND_OK，是否重试发送
    private boolean retryAnotherBrokerWhenNotStoreOK = false;

    # 客户端限制的消息大小，超过报错，同时服务端也会限制
    private int maxMessageSize = 1024 * 1024 * 4; // 4M

```

# 2.发送消息 #

## 2.1 构造Message ##
发送消息，第一步构造消息对象 Message。

```
public Message(String topic, String tags, String keys, int flag, byte[] body, boolean waitStoreMsgOK) {
        this.topic = topic;
        this.flag = flag;
        this.body = body;

        if (tags != null && tags.length() > 0)
            this.setTags(tags);

        if (keys != null && keys.length() > 0)
            this.setKeys(keys);

        this.setWaitStoreMsgOK(waitStoreMsgOK);
    }
```
　　一个应用尽可能用一个 Topic，消息子类型用 tags 来标识，tags 可以由应用自由设置。只有发送消息设置了tags，消费方在订阅消息时，才可以利用 tags 在 broker 做消息过滤。

　　每个消息在业务局面的唯一标识码，要设置到 keys 字段，方便将来定位消息丢失问题。服务器会为每个消息创建索引（哈希索引），应用可以通过 topic，key 来查询返条消息内容，以及消息被谁消费。由于是哈希索引，请务必保证 key 尽可能唯一，返样可以避免潜在的哈希冲突。
> 如在订单业务中，可以将订单号设置进去：
```
	String orderId = "20034568923546";
	message.setKeys(orderId);
```
将具体的消息放到 body 字段中，waitStoreMsgOK标识是否等待Broker处存放消息ok.


## 2.2 发送消息 ##
构造好Message消息后，调用 producer.send()方法完成对消息的发送。
底层调用：
```
 public SendResult send(Message msg,
        long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
    }
```
![](http://www.iocoder.cn/images/RocketMQ/2017_04_18/02.png)

步骤：获取消息路由信息，选择要发送到的消息队列，执行消息发送核心方法，并对发送结果进行封装返回。

```
private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // 校验 Producer 处于运行状态(ServiceState.RUNNING)
        this.makeSureStateOK();
        // 校验消息格式
        Validators.checkMessage(msg, this.defaultMQProducer);

        // 调用编号,调用时间,用于下面打印日志,标记为同义词发送消息,为监控平台链路追踪
        final long invokeID = random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;

        // 从namesrv中获取Topic的路由信息.
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false; // 判断是否发送超时
            MessageQueue mq = null;      // 最后选择消息要发送的队列
            Exception exception = null;
            SendResult sendResult = null; // 最后一次的发送结果.
            // 判断发送模式是否是同步,同步的话可能需要发送多次.
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;  // 第几次发送.
            String[] brokersSent = new String[timesTotal]; // 存储每次发送消息选择的broker名
            // 循环调用发送消息，直到成功
            for (; times < timesTotal; times++) {
                String lastBrokerName = null == mq ? null : mq.getBrokerName();
                // 选择要发送的消息队列.
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                    mq = mqSelected;
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        long costTime = beginTimestampPrev - beginTimestampFirst;
                        if (timeout < costTime) {
                            callTimeout = true;
                            break;
                        }

                        // 调用发送消息的核心方法.
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        endTimestamp = System.currentTimeMillis();
                        // 更新broker的可用性信息.
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            case SYNC:
                                // 同步发送成功但存储有问题时 && 配置存储异常时重新发送开关 时，进行重试
                                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                        continue;
                                    }
                                }

                                return sendResult;
                            default:
                                break;
                        }
                    } catch (RemotingException e) { // 打印异常，更新Broker可用性信息，更新继续循环
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } catch (MQClientException e) { // 打印异常，更新Broker可用性信息，继续循环
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } catch (MQBrokerException e) { // 打印异常，更新Broker可用性信息，部分情况下的异常，直接返回，结束循环
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        switch (e.getResponseCode()) {
                            case ResponseCode.TOPIC_NOT_EXIST:
                            case ResponseCode.SERVICE_NOT_AVAILABLE:
                            case ResponseCode.SYSTEM_ERROR:
                            case ResponseCode.NO_PERMISSION:
                            case ResponseCode.NO_BUYER_ID:
                            case ResponseCode.NOT_IN_CURRENT_UNIT:
                                continue;
                            default:
                                if (sendResult != null) {
                                    return sendResult;
                                }

                                throw e;
                        }
                    } catch (InterruptedException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());

                        log.warn("sendKernelImpl exception", e);
                        log.warn(msg.toString());
                        throw e;
                    }
                } else {
                    break;
                }
            }

            // 返回发送结果
            if (sendResult != null) {
                return sendResult;
            }

            // 根据不同情况，抛出不同的异常
            String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
                times,
                System.currentTimeMillis() - beginTimestampFirst,
                msg.getTopic(),
                Arrays.toString(brokersSent));

            info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

            MQClientException mqClientException = new MQClientException(info, exception);
            if (callTimeout) {
                throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
            }

            if (exception instanceof MQBrokerException) {
                mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
            } else if (exception instanceof RemotingConnectException) {
                mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
            } else if (exception instanceof RemotingTimeoutException) {
                mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
            } else if (exception instanceof MQClientException) {
                mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
            }

            throw mqClientException;
        }

        // Namesrv找不到异常
        List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
        if (null == nsList || nsList.isEmpty()) {
            throw new MQClientException(
                "No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NO_NAME_SERVER_EXCEPTION);
        }

        // 消息路由找不到异常
        throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
            null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
    }

```



# 3.消息发送结果处理 #

　　消息发送成功或者失败，要打印消息日志，务必要打印 sendresult 和 key 字段。
send 消息方法，只要不抛异常，就代表发送成功。但是发送成功会有多个状态，在 sendResult 里定义。

```
public enum SendStatus {
    // 消息发送成功
    SEND_OK,
    // 消息发送成功，但是服务器刷盘超时，消息已经进入服务器队列，只有此时服务器宕机，消息才会丢失
    FLUSH_DISK_TIMEOUT,
    // 消发发送成功，但是服务器同步到 Slave 时超时，消息已经迕入服务器队列，只有此时服务器宕机，消息才会丢失
    FLUSH_SLAVE_TIMEOUT,
    // 消息发送成功，但是此时 slave 不可用，消息已经进入服务器队列，只有此时服务器宕机，消息才会丢失
    SLAVE_NOT_AVAILABLE,
}
```

# 4.发送形式 #
```
/**
 * 消息发送模式
 */
public enum CommunicationMode {
    SYNC,   //同步.
    ASYNC,  //异步.
    ONEWAY, //只管发送
}
```

- SYNC
```
producer.send(msg)
```
> 　　同步的发送方式，会等待发送结果后才返回。可以用 send(msg, timeout) 的方式指定等待时间，如果不指定，就是默认的 3000ms. 这个timeout 最终会被设置到 ResponseFuture 里，在发送完消息后，用 countDownLatch 去 await timeout的时间，如果过期，就会抛出异常。


- ASYNC
```
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        System.out.printf("%-10d OK %s %n", index, sendResult.getMsgId());
    }
    @Override
    public void onException(Throwable e) {
        System.out.printf("%-10d Exception %s %n", index, e);
        e.printStackTrace();
    }
});
```

> 　　异步的发送方式，发送完后，立刻返回。Client 在拿到 Broker 的响应结果后，会回调指定的 callback. 这个 API 也可以指定 Timeout，不指定也是默认的 3000ms.

- ONEWAY

```
producer.sendOneway(msg);
```
> 一个 RPC 调用，通常是这样一个过程
> 1. 客户端发送请求到服务器
> 2. 服务器处理该请求
> 3. 服务器吐客户端返回应答
> 
>　　 所以一个 RPC 的耗时时间是上述三个步骤的总和，而某些场景要求耗时非常短，但是对可靠性要求并不高，例如
> 日志收集类应用，此类应用可以采用 oneway 形式调用，oneway 形式只发送请求并等待应答，而发送请求在客
> 户端实现局面仅仅是一个 os 系统调用的开销，即将数据写入客户端的 socket 缓冲区，此过程耗时通常在微秒级。

