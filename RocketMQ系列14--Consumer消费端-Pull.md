---
title:  RocketMQ系列14--Consumer消费端-Pull
date:  2018-09-07
categories:  RocketMQ 
tags: [rocketmq,Producer,Consumer,Pull] 
	 
---


# 0.前言 #

push和pull模式在上一篇已经说过了。push是broker主动去向consumer推送消息，他们之间维持者一个长连接，从而实现broker向消费者推送消息。而本节则讨论拉模式 DefaultPullConsumer的实现。

RocketMQ拉模式，消费者不自动向消息服务器拉取消息，而是将控制权移交给应用程序，RocketMQ消费者只是提供拉取消息API。


# 1.pull消费端代码 #

```
public class PullConsumer {
    private static final Map<MessageQueue, Long> OFFSE_TABLE = new HashMap<MessageQueue, Long>();

    public static void main(String[] args) throws MQClientException {
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("please_rename_unique_group_name_5");
        consumer.setNamesrvAddr("192.168.81.132:9876;192.168.81.134:9876");
        consumer.start();

        Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest1");

        for (MessageQueue mq : mqs) {
            System.out.printf("Consume from the queue: %s%n", mq);
            SINGLE_MQ:
            while (true) {
                try {
                    PullResult pullResult =
                        consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                    System.out.printf("%s%n", pullResult);
                    //在本地创建offseTable:Map<MessageQueue, Long>变量，根据上一步返回的结果，
					//将该MessageQueue的下一次消费开始位置记录下来
                    putMessageQueueOffset(mq, pullResult.getNextBeginOffset());
                    switch (pullResult.getPullStatus()) {
                        case FOUND:
                            break;
                        case NO_MATCHED_MSG:
                            break;
                        case NO_NEW_MSG:
                            break SINGLE_MQ;
                        case OFFSET_ILLEGAL:
                            break;
                        default:
                            break;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }

        consumer.shutdown();
    }

    private static long getMessageQueueOffset(MessageQueue mq) {
        Long offset = OFFSE_TABLE.get(mq);
        if (offset != null)
            return offset;

        return 0;
    }

    private static void putMessageQueueOffset(MessageQueue mq, long offset) {
        OFFSE_TABLE.put(mq, offset);
    }

}
```

# 2.DefaultMQPullConsumer的启动 #

```
public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // 检查配置信息
                this.checkConfig();
                // 复制订阅信息
                this.copySubscription();

                if (this.defaultMQPullConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                    this.defaultMQPullConsumer.changeInstanceNameToPID();
                }
                // 创建 MQClientInstance 对象.
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPullConsumer, this.rpcHook);

                // 设置消费端负载均衡配置.
                this.rebalanceImpl.setConsumerGroup(this.defaultMQPullConsumer.getConsumerGroup());
                this.rebalanceImpl.setMessageModel(this.defaultMQPullConsumer.getMessageModel());
                this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPullConsumer.getAllocateMessageQueueStrategy());
                this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

				// 构造PullAPIWrapper
                this.pullAPIWrapper = new PullAPIWrapper(
                    mQClientFactory,
                    this.defaultMQPullConsumer.getConsumerGroup(), isUnitMode());
                this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

                // 获取 offsetStore
                if (this.defaultMQPullConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPullConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPullConsumer.getMessageModel()) {
                        case BROADCASTING:
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPullConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPullConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                    this.defaultMQPullConsumer.setOffsetStore(this.offsetStore);
                }
                // 加载 offsetStore 进度条.
                this.offsetStore.load();

                // 注册该消费者.
                boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPullConsumer.getConsumerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;

                    throw new MQClientException("The consumer group[" + this.defaultMQPullConsumer.getConsumerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

                mQClientFactory.start();
                log.info("the consumer [{}] start OK", this.defaultMQPullConsumer.getConsumerGroup());
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The PullConsumer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }

    }
```




一开始的serverState的状态自然为CREAT_JUST，调用checkConfig()，其中先是对ConsumerGroup进行验证，非空，合法(符合正则规则，且长度不超过配置最大值)，且不为默认值(防止消费者集群名冲突)，然后对消费者消息模式、消息队列分配算法进行非空、合法校验。

调用copySubscription()方法，将配置在DefaultMQPullConsumer中的topic信息构造成subscriptionData数据结构，以topic为key以subscriptionData为value以键值对形式存到rebalanceImpl的subscriptionInner中。

接下来从MQCLientManager中得到MQClient的实例

再往后是对rebalanceImpl的配置，我们重点看下rebalanceImpl，它是在DefaultMQPullConsumerImpl成员中直接构造private RebalanceImpl rebalanceImpl = new RebalancePullImpl(this);即在DefaultMQPullConsumerImpl初始化的时候构造。接下来对其消费者组名、消息模式(默认集群)、队列分配算法(默认平均分配)、消费者客户端实例进行配置，配置信息都是从DefaultMQPullConsumer中取得。

初始化消费者的offsetStore，offset即偏移量，可以理解为消费进度，这里根据不同的消息模式来选择不同的策略。如果是广播模式，那么所有消费者都应该收到订阅的消息，那么每个消费者只应该自己消费的消费队列的进度，那么需要把消费进度即offsetStore存于本地采用LocalFileOffsetStroe，相反的如果是集群模式，那么集群中的消费者来平均消费消息队列，那么应该把消费进度存于远程采用RemoteBrokerOffsetStore。

调用相应的load方法加载。

将当前消费者注册在MQ客户端实例上之后，调用MQClientInstance的start()方法，启动消费者客户端

# 3.根据topic获取对应的MessageQueue（即可被订阅的队列） #

```
public Set<MessageQueue> fetchSubscribeMessageQueues(String topic) throws MQClientException {
        try {
            // 1. 从 namesrv 获取该topic对应的broker信息和topic配置信息.
            TopicRouteData topicRouteData = this.mQClientFactory.getMQClientAPIImpl().getTopicRouteInfoFromNameServer(topic, timeoutMillis);

            // 2. 遍历 TopicRouteData 对象中的 QueueData 列表中的每个QueueMessage的值.
            //    2.1 判断该 QueueData 对象是否有读权限,有权限的话则创建 MessageQueue对象,并构成MessageQueue集合,并返回.
            // 反走,抛出异常.
            if (topicRouteData != null) {
                Set<MessageQueue> mqList = MQClientInstance.topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
                if (!mqList.isEmpty()) {
                    return mqList;
                } else {
                    throw new MQClientException("Can not find Message Queue for this topic, " + topic + " Namesrv return empty", null);
                }
            }
        } catch (Exception e) {
            throw new MQClientException(
                "Can not find Message Queue for this topic, " + topic + FAQUrl.suggestTodo(FAQUrl.MQLIST_NOT_EXIST),
                e);
        }

        throw new MQClientException("Unknow why, Can not find Message Queue for this topic, " + topic, null);
    }
```
# 4.DefaultMQPullConsumer.pullBlockIfNotFound(...) #

```
public PullResult pullBlockIfNotFound(MessageQueue mq, String subExpression, long offset, int maxNums)
```
> mq:要消费的队列；
> subExpression：消费过滤规则
> offset：该messageQueue对象的开始消费位置
> maxNums：32，表示获取的最大消息个数。

该方法最终调用DefaultMQPullConsumerImpl.pullSyncImpl(MessageQueue mq, String subExpression, long offset, int maxNums, boolean block)方法。

```
    private PullResult pullSyncImpl(MessageQueue mq, String subExpression, long offset, int maxNums, boolean block,
        long timeout)
        throws MQClientException, RemotingException, MQBrokerException, InterruptedException {

        // 0.确保该消费者处于运行状态.
        this.makeSureStateOK();
        if (null == mq) {
            throw new MQClientException("mq is null", null);
        }
        if (offset < 0) {
            throw new MQClientException("offset < 0", null);
        }
        if (maxNums <= 0) {
            throw new MQClientException("maxNums <= 0", null);
        }

        // 1.topic是否在RebalanceImpl.subscriptionInner:ConcurrentHashMap<String,SubscriptionData>变量中;
        // 不存在话,构造SubscriptionData对象保存到RebalanceImpl.subscriptionInner变量中
        this.subscriptionAutomatically(mq.getTopic());

        // 2.构建消息的标志位sysFlag，其中suspend和subscription为true（即该标记位的第2/3位为1），其他commit和classFilter两位为false（第1/4位为0）
        int sysFlag = PullSysFlag.buildSysFlag(false, block, true, false);

        // 3.构造SubscriptionData对象并返回
        SubscriptionData subscriptionData;
        try {
            subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPullConsumer.getConsumerGroup(),
                mq.getTopic(), subExpression);
        } catch (Exception e) {
            throw new MQClientException("parse subscription error", e);
        }

        long timeoutMillis = block ? this.defaultMQPullConsumer.getConsumerTimeoutMillisWhenSuspend() : timeout;

        // 4.从Broker拉取消息内容.
        PullResult pullResult = this.pullAPIWrapper.pullKernelImpl(
            mq,
            subscriptionData.getSubString(),
            0L,
            offset,
            maxNums,
            sysFlag,
            0,
            this.defaultMQPullConsumer.getBrokerSuspendMaxTimeMillis(),
            timeoutMillis,
            CommunicationMode.SYNC,
            null
        );

        // 5.对拉取消息的响应结果进行处理，主要是消息反序列化
        this.pullAPIWrapper.processPullResult(mq, pullResult, subscriptionData);
        if (!this.consumeMessageHookList.isEmpty()) {
            ConsumeMessageContext consumeMessageContext = null;
            consumeMessageContext = new ConsumeMessageContext();
            consumeMessageContext.setConsumerGroup(this.groupName());
            consumeMessageContext.setMq(mq);
            consumeMessageContext.setMsgList(pullResult.getMsgFoundList());
            consumeMessageContext.setSuccess(false);
            this.executeHookBefore(consumeMessageContext);
            consumeMessageContext.setStatus(ConsumeConcurrentlyStatus.CONSUME_SUCCESS.toString());
            consumeMessageContext.setSuccess(true);
            this.executeHookAfter(consumeMessageContext);
        }
        return pullResult;
    }
```

# 5.获取消费进度 fetchConsumeOffset（...）#
调用DefaultMQPullConsumer.fetchConsumeOffset(MessageQueue mq, boolean fromStore)方法获取MessageQueue队列的消费进度，其中fromStore为true表示从存储端（即Broker端）获取消费进度；若fromStore为false表示从本地内存获取消费进度；

1、对于从存储端获取消费进度（即fromStore=true）的情况：

1.1)对于LocalFileOffsetStore对象，从本地加载offsets.json文件，然后获取该MessageQueue对象的offset值；

1.2)对于RemoteBrokerOffsetStore对象,获取逻辑如下：

A）以MessageQueue对象的brokername从MQClientInstance. brokerAddrTable中获取Broker的地址；若没有获取到则立即调用updateTopicRouteInfoFromNameServer方法然后再次获取；

B）构造QueryConsumerOffsetRequestHeader对象，其中包括topic、consumerGroup、queueId；然后调用MQClientAPIImpl.queryConsumerOffset (String addr, QueryConsumerOffsetRequestHeader requestHeader, long timeoutMillis)方法向Broker发送QUERY_CONSUMER_OFFSET请求码，获取消费进度Offset；

C）用上一步从Broker获取的offset更新本地内存的消费进度列表数据RemoteBrokerOffsetStore.offsetTable:ConcurrentHashMap<MessageQueue, AtomicLong>变量值；

D）返回该offset值；

2、对于从本地内存获取消费进度（即fromStore=false）的情况：

对于LocalFileOffsetStore或者RemoteBrokerOffsetStore对象，均是以MessageQueue对象作为key值从各自对象的offsetTable变量中获取相应的消费进度。

# 6.推荐 #
【1】 [RocketMQ——Consumer篇：PULL模式下的消息消费（DefaultMQPullConsumer）](https://blog.csdn.net/meilong_whpu/article/details/77081676)















