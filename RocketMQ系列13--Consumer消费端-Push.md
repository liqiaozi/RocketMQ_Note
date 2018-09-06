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




# 1.RocketMQ消息订阅及长轮询机制 #

同样，RocketMQ消息订阅有两种模式，一种是Push模式，即MQServer主动向消费端推送；另外一种是Pull模式，即消费端在需要时，主动到MQServer拉取。通过研究源码可知，RocketMQ的消费方式都是基于拉模式拉取消息的。
那么在RocketMQ中如何解决以做到高效的消息消费呢？

答案就是在rocketmq中有一种长轮询机制（对普通轮询的一种优化），来平衡上面push/pull模型的各自缺点。基本设计思路：

消费者端如果第一次尝试pull消息失败（比如：broker端没有消息可以被消费），这时并不立即给消费者端返回response的响应，而是先hold住并挂起该请求（将该请求保存至pullRequestTable本地缓存变量中），然后broker端的后台独立线程（PullRequestHoldService）会从pullRequestTable本地缓存变量中不断去取信息（查询待拉取消息的偏移量是否小于消费队列的最大偏移量，如果条件成立则说明有新消息到达broker端）。同时,另外一个ReputMessageService线程不断的构建ConsumerQueue/IndexFile数据，并取出hold住的pull请求进行二次处理。如果有新消息到达broker端，则通过重新调用一次业务处理器-pullmessageProcessor的处理请求方法-processRequest来重新尝试拉取消息。

# 2.demo代码 #

push模式下的消费者端代码.
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

        consumer.setConsumeMessageBatchMaxSize(10);

        consumer.registerMessageListener(new MessageListenerConcurrently() {

            // 接收消息.
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,ConsumeConcurrentlyContext context) {
                for(MessageExt msg : msgs){
                    try {
                        String topic = msg.getTopic();
                        String tags = msg.getTags();
                        String msgBody =  new String(msg.getBody(),"utf-8");
                        System.out.println("收到消息--" + " topic:" + topic + " ,tags:" + tags + " ,msg:" +msgBody);
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();

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
当使用 push模式的消费者时,只需要在消费者端注册一个监听器(new MessageListenerConcurrently),Consumer在收到消息后主动调用这个监听器完成消费并进行自己业务逻辑处理.


# 3.Push方式的启动流程分析 #

![](https://upload-images.jianshu.io/upload_images/11560519-67bc66230732b52f.png)
```
public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                    this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
                this.serviceState = ServiceState.START_FAILED;
                // 检查配置信息.
                this.checkConfig();
                // 复制订阅信息.
                this.copySubscription();

                if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                    this.defaultMQPushConsumer.changeInstanceNameToPID();
                }

                // 创建 MQClientInstance 对象.
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

                // 设置负载均衡示例 rebalanceImpl 的属性值.
                this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
                this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
                this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
                this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

                // 初始化 pullAPIWrapper
                this.pullAPIWrapper = new PullAPIWrapper(
                    mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
                this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

                // 根据配置初始化 offsetStore 并加载进度条.
                if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPushConsumer.getMessageModel()) {
                        case BROADCASTING:
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
                }
                this.offsetStore.load();

                // 初始化消息消费服务并且启动.
                if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                    this.consumeOrderly = true;
                    this.consumeMessageService =
                        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
                } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                    this.consumeOrderly = false;
                    this.consumeMessageService =
                        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
                }
                this.consumeMessageService.start();

                boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    this.consumeMessageService.shutdown();
                    throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

                mQClientFactory.start();
                log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }

        // 更新订阅主题信息.
        this.updateTopicSubscribeInfoWhenSubscriptionChanged();
        // 检查broker上的所有客户端.
        this.mQClientFactory.checkClientInBroker();
        // 枷锁并向所有的broker端发送心跳.
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        // 重新负载均衡.
        this.mQClientFactory.rebalanceImmediately();
    }
```

```
public void start() throws MQClientException {

        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    // 路由到namesrv 地址.
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.mQClientAPIImpl.fetchNameServerAddr();
                    }
                    // Start request-response channel
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    this.startScheduledTask();
                    // Start pull service
                    this.pullMessageService.start();
                    // Start rebalance service
                    this.rebalanceService.start();
                    // Start push service
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case RUNNING:
                    break;
                case SHUTDOWN_ALREADY:
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
    }

```

# 6.参考 #
[1][ 消息中间件—RocketMQ消息消费（一）](https://www.jianshu.com/p/f071d5069059)

[2] [分布式开放消息系统(RocketMQ)的原理与实践](https://www.jianshu.com/p/453c6e7ff81c)

















