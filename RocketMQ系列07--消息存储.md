---
title:  RocketMQ系列07--消息存储
date:  2018-09-09
categories:  RocketMQ 
tags: [rocketmq,消息存储] 
	 
---

# 0. RocketMQ消息存储 #

rocketmq的broker消息存储主要包括3个部分,分别是:

commitLog的存储,
consumerQueue的存储,
index的存储.

# 1. Consumer Queue #
consumer queue是消息的逻辑队列,相当于字典的目录,用来指定消息在物理文件commit log上的位置。

我们可以在配置中指定 consumerqueue 与 commitlog存储的目录。

每个topic下的每个queue都有一个对应的 consumerqueue文件。


    ${rocketmq.home}/store/consumequeue/${topicName}/${queueId}/${fileName}


![](https://upload-images.jianshu.io/upload_images/11560519-e9d1580352721a01.png)

**Consume Queue文件组织示意图**

> 1.根据topic和queueId来组织文件，图中TopicA有两个队列0,1，那么TopicA和QueueId=0组成一个ConsumeQueue，TopicA和QueueId=1组成另一个ConsumeQueue。
> 
> 2.按照消费端的GroupName来分组重试队列，如果消费端消费失败，消息将被发往重试队列中，比如图中的%RETRY%ConsumerGroupA。
> 
> 3.按照消费端的GroupName来分组死信队列，如果消费端消费失败，并重试指定次数后，仍然失败，则发往死信队列，比如图中的%DLQ%ConsumerGroupA。

Consume Queue中存储单元是一个20字节定长的二进制数据，顺序写顺序读，如下图所示：

![](http://upload-images.jianshu.io/upload_images/175724-7212acc81b91c086.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- CommitLog Offset是指这条消息在CommitLog中的实际偏移量；
- Size 存储消息的大小
- Message Tag Hashcode 存储消息的Tag哈希值：主要用于订阅时消息过滤（订阅时如果指定了Tag，会根据HashCode来快速查找到订阅的消息。）

# 2. CommitLog #
消息存放的物理文件，每台broker上的commitlog被本机所有的queue共享，不做任何区分。
文件的默认位置如下，可以通过配置文件配置修改：

    ${user.home} \store\${commitlog}\${fileName}

CommitLog的消息存储单元长度不固定，文件顺序写，随机读。消息的存储结构如下表所示，按照编号顺序以及编号对应的内容依次存储。
![](http://upload-images.jianshu.io/upload_images/175724-96ed677eb504abfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



# 3.消息的索引文件index #

如果一个消息包含key值的话，会使用IndexFile存储消息索引，文件内容结构如图：

![](http://upload-images.jianshu.io/upload_images/175724-4deee0fb9d08e02d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/175724-4deee0fb9d08e02d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

索引文件主要用于根据key来查询消息的，流程主要是：

1.根据查询的key的 (hash值 % slotNum) 得到具体的槽的位置（slotNum）是一个索引文件里面包含的最大槽的数目。

2.根据 slotValue(slot 位置对应的值)查找到索引项列表的最后一项(倒序排列,slotValue 总是指向最新的一个索引项)

3.遍历索引项列表返回查询时间范围内的结果集(默认一次最大返回的 32 条记录) .

# 4.源码解读 #

入口类：SendMessageProcessor.java
```
    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                          RemotingCommand request) throws RemotingCommandException {
        SendMessageContext mqtraceContext;
        switch (request.getCode()) {
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.consumerSendMsgBack(ctx, request);
            default:
                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
                if (requestHeader == null) {
                    return null;
                }

                mqtraceContext = buildMsgContext(ctx, requestHeader);
                this.executeSendMessageHookBefore(ctx, request, mqtraceContext);

                RemotingCommand response;
                if (requestHeader.isBatch()) {
                    response = this.sendBatchMessage(ctx, request, mqtraceContext, requestHeader);
                } else {
                    response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);
                }

                this.executeSendMessageHookAfter(response, mqtraceContext);
                return response;
        }
    }
```

![](https://upload-images.jianshu.io/upload_images/6302559-124bde9f2799a151.png)




