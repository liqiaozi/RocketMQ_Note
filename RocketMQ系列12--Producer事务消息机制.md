---
title:  RocketMQ系列12--Producer 事务消息
date:  2018-09-01
categories:  RocketMQ 
tags: [rocketmq,Producer,事务消息] 
	 
---

# 0. 前言 #
RocketMQ除了支持普通消息，顺序消息，另外还支持事务消息。首先讨论一下什么是事务消息以及支持事务消息的必要性。我们以一个转帐的场景为例来说明这个问题：Bob向Smith转账100块。

[转账业务](https://www.jianshu.com/p/453c6e7ff81c)


# 1. 事务消息 #
通过消息的异步事务,可以保证本地事务和消息发送同时执行成功或者失败,从而保证了数据的最终一致性.

rocketmq事务消息是发生在 producer和broker之间,是事务的二阶段提交，如图：


![](http://lifestack.cn/wp-content/uploads/2015/09/%E4%BA%8B%E5%8A%A1%E9%80%BB%E8%BE%91.jpg)

只有在消息发送成功，并且本地操作执行成功时，才发送提交事务消息，做事务提交；

其他的情况，例如消息的发送失败，则直接发送回滚消息，进行回滚，或者发送消息成功，但是本地执行操作失败，也是发送回滚消息，进行回滚。

阶段解读：

**一阶段(步骤 1 2 3)**:Producer 向Broker发送一条 PROPERTY_TRANSACTION_PREPARED 的消息，Broker接收到消息后保存在 CommitLog中，然后返回结果给 Producer,但是该类型的消息对Consumer是不可见的，Consumer无法消费。因为该类型消息在保存的时候，commitLogOffset没有被保存到 consumerQueue 中，所以客户端通过 consumerQueue 取不到 consumerQueue,所以无法被消费。

**二阶段(步骤 4 5)**: Producer端的 tranExecuter.executeLocalTransaction 执行本地操作，返回本地事务的状态，然后发送一条类型为TransactionCommitType或者TransactionRollbackType的消息到Broker确认提交或者回滚。
Broker通过Request中的commitLogOffset，获取到上面状态为TransactionPreparedType的消息（简称消息A），然后重新构造一条与消息A内容相同的消息B，设置状态为TransactionCommitType或者TransactionRollbackType，然后保存。其中TransactionCommitType类型的，会放commitLogOffset到consumerQueue中，TransactionRollbackType类型的，消息体设置为空，不会放commitLogOffset到consumerQueue中。

所以，整个事务消息过程中，broker一共保存2条消息。

# 2. producer端 #

在最新版本 4.4.0-SNAPSHOT 中，TransactionMQProducer已经把 本地执行事务的代码 和 broker回查 producer本地事务的方法封装到了一个监听器中TransactionListener。

```
public class TransactionMQProducer extends DefaultMQProducer {
    private TransactionListener transactionListener;

    private ExecutorService executorService;

	...
}

```
```
public interface TransactionListener {
    /**
     * When send transactional prepare(half) message succeed, this method will be invoked to execute local transaction.
     *
     * @param msg Half(prepare) message
     * @param arg Custom business parameter
     * @return Transaction state
     */
    LocalTransactionState executeLocalTransaction(final Message msg, final Object arg);

    /**
     * When no response to prepare(half) message. broker will send check message to check the transaction status, and this
     * method will be invoked to get local transaction status.
     *
     * @param msg Check message
     * @return Transaction state
     */
    LocalTransactionState checkLocalTransaction(final MessageExt msg);
}
```

```
public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        // 本地事务执行逻辑及broker回查事务状态实现类.
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("Transaction_MQProducer_Group");
        //设置 nameserver地址.
        producer.setNamesrvAddr("192.168.81.132:9876;192.168.81.134:9876");

        //设置broker回查prodducer的并发数
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg = new Message("TopicTest1234", 
                        tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                
                SendResult sendResult = producer.sendMessageInTransaction(msg, i);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}

```
 根据业务需要，自己实现 TransactionListener接口中的方法。
```
public class TransactionListenerImpl implements TransactionListener {
    private AtomicInteger transactionIndex = new AtomicInteger(0);

    private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        int value = transactionIndex.getAndIncrement();
        int status = value % 3;
        localTrans.put(msg.getTransactionId(), status);
        return LocalTransactionState.UNKNOW;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        Integer status = localTrans.get(msg.getTransactionId());
        if (null != status) {
            switch (status) {
                case 0:
                    return LocalTransactionState.UNKNOW;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                case 2:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
            }
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```

# 3. consumer端 #
同普通消息消费一样。




# 4. 发送消息详解 #
```
public TransactionSendResult sendMessageInTransaction(final Message msg,
                                                          final TransactionListener tranExecuter, final Object arg)
        throws MQClientException {

        // 判断是否有本地事务监听器
        if (null == tranExecuter) {
            throw new MQClientException("tranExecutor is null", null);
        }
        // 校验message信息.
        Validators.checkMessage(msg, this.defaultMQProducer);

        // 标记该消息为 事务消息.
        SendResult sendResult = null;
        MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
        MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
        // 第一次发送消息.
        try {
            sendResult = this.send(msg);
        } catch (Exception e) {
            throw new MQClientException("send message Exception", e);
        }

        LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
        Throwable localException = null;
        switch (sendResult.getSendStatus()) {
            case SEND_OK: {
                try {
                    if (sendResult.getTransactionId() != null) {
                        msg.putUserProperty("__transactionId__", sendResult.getTransactionId());
                    }
                    String transactionId = msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                    if (null != transactionId && !"".equals(transactionId)) {
                        // 将 transactionId 设置到消息中.
                        msg.setTransactionId(transactionId);
                    }
                    // 执行本地事务.*********重要.
                    // 判断 localTransactionState 是否是 COMMIT_MESSAGE状态.
                    localTransactionState = tranExecuter.executeLocalTransaction(msg, arg);
                    if (null == localTransactionState) {
                        localTransactionState = LocalTransactionState.UNKNOW;
                    }

                    if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
                        log.info("executeLocalTransactionBranch return {}", localTransactionState);
                        log.info(msg.toString());
                    }
                } catch (Throwable e) {
                    log.info("executeLocalTransactionBranch exception", e);
                    log.info(msg.toString());
                    localException = e;
                }
            }
            break;
            case FLUSH_DISK_TIMEOUT:
            case FLUSH_SLAVE_TIMEOUT:
            case SLAVE_NOT_AVAILABLE:
                localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
                break;
            default:
                break;
        }

        try {
			// 第二次发送消息.
            this.endTransaction(sendResult, localTransactionState, localException);
        } catch (Exception e) {
            log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
        }

        // 构造返回信息 TransactionSendResult
        TransactionSendResult transactionSendResult = new TransactionSendResult();
        transactionSendResult.setSendStatus(sendResult.getSendStatus());
        transactionSendResult.setMessageQueue(sendResult.getMessageQueue());
        transactionSendResult.setMsgId(sendResult.getMsgId());
        transactionSendResult.setQueueOffset(sendResult.getQueueOffset());
        transactionSendResult.setTransactionId(sendResult.getTransactionId());
        transactionSendResult.setLocalTransactionState(localTransactionState);
        return transactionSendResult;
    }

```

```
public void endTransaction(
        final SendResult sendResult,
        final LocalTransactionState localTransactionState,
        final Throwable localException) throws RemotingException, MQBrokerException, InterruptedException, UnknownHostException {
        // 得到message的地址信息
        final MessageId id;
        if (sendResult.getOffsetMsgId() != null) {
            id = MessageDecoder.decodeMessageId(sendResult.getOffsetMsgId());
        } else {
            id = MessageDecoder.decodeMessageId(sendResult.getMsgId());
        }
        // 得到事务消息的Id
        String transactionId = sendResult.getTransactionId();
        // 根据brokerName 找到broker的地址.
        final String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(sendResult.getMessageQueue().getBrokerName());

        // 构造结束事务请求头信息.
        EndTransactionRequestHeader requestHeader = new EndTransactionRequestHeader();
        requestHeader.setTransactionId(transactionId);
        requestHeader.setCommitLogOffset(id.getOffset());
        switch (localTransactionState) {
            case COMMIT_MESSAGE:
                requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
                break;
            case ROLLBACK_MESSAGE:
                requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
                break;
            case UNKNOW:
                requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
                break;
            default:
                break;
        }

        requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
        requestHeader.setTranStateTableOffset(sendResult.getQueueOffset());
        requestHeader.setMsgId(sendResult.getMsgId());
        String remark = localException != null ? ("executeLocalTransactionBranch exception: " + localException.toString()) : null;

        // oneway 形式发送事务应答信息.
        this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, requestHeader, remark,
            this.defaultMQProducer.getSendMsgTimeout());
    }
```




# 5. 推荐及参考 #

【1】 [事务消息](https://yq.aliyun.com/articles/55630)

【2】 [分布式开放消息系统(RocketMQ)的原理与实践](https://www.jianshu.com/p/453c6e7ff81c)