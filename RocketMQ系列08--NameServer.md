---
title:  RocketMQ系列08--Namersrv
date:  2018-09-09
categories:  RocketMQ 
tags: [rocketmq,namesrv] 
	 
---

# 1.前言 #

主要管理所有的broker信息,让producer和consumer都能获取到正确的broker信息,进行业务处理.类似于zookeeper的服务治理中心.简而言之,主要包含下面几方面的功能:

1. Broker启动的时候会向NameSrv发送注册请求，Namesrv接收broker的请求注册路由信息和保存活跃的broker列表，包括master和slave;
2. 用来保存所有的topic和该topic所有队列的列表；
3. Namesrv用来保存所有broker的Filter列表；
4. 接收producer和consumer的请求，根据某个topic获取到broker的路由信息。


# 2.namesrv启动和初始化 #
namesrv启动的时候主要涉及 NamesrvStartup/NamesrvController 两个类。NamesrvStartup 负责解析命令行的一些参数到各种 Config 对象中（NamesrvConfig/NettyServerConfig等），如果命令行参数中带有配置文件的路径，也会从配置文件中读取配置到各种 Config 对象中，然后初始化 NamesrvController，配置shutdownHook, 启动 NamesrvController。
 
NamesrvController 会去初始化和启动各个组件，主要是:
- 创建NettyServer，注册 requestProcessor，用于处理不同的网络请求
- 启动 NettyServer
- 启动各种 scheduled task.

```
public static NamesrvController main0(String[] args) {

        try {
            // 加载命令行配置信息.初始化NamesrvController配置信息.
            NamesrvController controller = createNamesrvController(args);
            start(controller);
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }
```

```
 public static NamesrvController start(final NamesrvController controller) throws Exception {

        if (null == controller) {
            throw new IllegalArgumentException("NamesrvController is null");
        }
        // 初始化 配置信息
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }

        Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                controller.shutdown();
                return null;
            }
        }));

        controller.start();

        return controller;
    }
```

```
public boolean initialize() {
        // 加载NameServer的配置参数，将配置参数加载保存到一个HashMap中
        this.kvConfigManager.load();
        // 初始化 BrokerHousekeepingService 对象为参数初始化
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        // 注册默认的处理类 DefaultRequestProcessor,所有的请求均由该处理类的 processRequest 方法来处理
        this.registerProcessor();

        // 设置两个定时任务
        // 每隔十秒钟遍历brokerLiveTable集合，查看每个broker的最后更新时间是否超过了两分钟，
        // 超过则关闭broker的渠道并清理 RouteInfoManager 类的topicQueueTable、 brokerAddrTable、 clusterAddrTable、 filterServerTable成员变量
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        // 每隔 10 分钟打印一次 NameServer 的配置参数。即KVConfigManager.configTable 变量的内容
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);

        // 启动 NameServer 的 Netty 服务端（ NettyRemotingServer），监听渠道的请求信息。
        // 当收到客户端的请求信息之后会初始化一个线程，并放入线程池中进行处理,该线程调用 DefaultRequestProcessor. processRequest 方法来处理请求
        if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
            // Register a listener to reload SslContext
            try {
                fileWatchService = new FileWatchService(
                    new String[] {
                        TlsSystemConfig.tlsServerCertPath,
                        TlsSystemConfig.tlsServerKeyPath,
                        TlsSystemConfig.tlsServerTrustCertPath
                    },
                    new FileWatchService.Listener() {
                        boolean certChanged, keyChanged = false;
                        @Override
                        public void onChanged(String path) {
                            if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                                log.info("The trust certificate changed, reload the ssl context");
                                reloadServerSslContext();
                            }
                            if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                                certChanged = true;
                            }
                            if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                                keyChanged = true;
                            }
                            if (certChanged && keyChanged) {
                                log.info("The certificate and private key changed, reload the ssl context");
                                certChanged = keyChanged = false;
                                reloadServerSslContext();
                            }
                        }
                        private void reloadServerSslContext() {
                            ((NettyRemotingServer) remotingServer).loadSslContext();
                        }
                    });
            } catch (Exception e) {
                log.warn("FileWatchService created error, can't load the certificate dynamically");
            }
        }

```


# 3.处理broker的注册请求 # 
DefaultRequestProcessor.processRequest处理请求, 如果 request.getCode 是 RequestCode.REGISTER_BROKER, 就去注册。这里会根据request.version来判断，从V3_0_11 开始支持了FilterServer。

```
case RequestCode.REGISTER_BROKER:
Version brokerVersion = MQVersion.value2Version(request.getVersion());
if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
    return this.registerBrokerWithFilterServer(ctx, request);
} else {
    return this.registerBroker(ctx, request);
}
```

```
  public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {
        RegisterBrokerResult result = new RegisterBrokerResult();
        try {
            try {
                this.lock.writeLock().lockInterruptibly();

                /**
                 * 维护 RouteInfoManager.clusterAddrTable 变量； 若 Broker 集群名字不
                 在该 Map 变量中，则初始化一个 Set 集合，将 brokerName 存入该 Set 集合中，
                 然后以 clusterName 为 key 值，该 Set 集合为 values 值存入此 Map 变量中
                 */
                Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
                if (null == brokerNames) {
                    brokerNames = new HashSet<String>();
                    this.clusterAddrTable.put(clusterName, brokerNames);
                }
                brokerNames.add(brokerName);

                boolean registerFirst = false;

                /**
                 * 维护 RouteInfoManager.brokerAddrTable 变量，该变量是维护 Broker
                 的名称、 ID、地址等信息的。 若该 brokername 不在该 Map 变量中，则创建
                 BrokerData 对象，该对象包含了 brokername，以及 brokerId 和 brokerAddr 为
                 K-V 的 brokerAddrs 变量；然后以 brokername 为 key 值将 BrokerData 对象存入
                 该 brokerAddrTable 变量中； 说明同一个 BrokerName 下面可以有多个不同
                 BrokerId 的 Broker 存在，表示一个 BrokerName 有多个 Broker 存在，通过
                 BrokerId 来区分主备
                 */
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null == brokerData) {
                    registerFirst = true;
                    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                    this.brokerAddrTable.put(brokerName, brokerData);
                }
                String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
                registerFirst = registerFirst || (null == oldAddr);

                // 若 Broker 的注册请求消息中 topic 的配置不为空，并且该 Broker 是主用（即 brokerId=0）
                if (null != topicConfigWrapper
                    && MixAll.MASTER_ID == brokerId) {
                    //则根据 NameServer 存储的 Broker 版本信息来判断是否需要更新 NameServer 端的 topic 配置信息
                    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                        || registerFirst) {
                        ConcurrentMap<String, TopicConfig> tcTable =
                            topicConfigWrapper.getTopicConfigTable();
                        if (tcTable != null) {
                            for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                                this.createAndUpdateQueueData(brokerName, entry.getValue());
                            }
                        }
                    }
                }

                //初始化 BrokerLiveInfo 对象并以 broker 地址为 key 值存入
                // brokerLiveTable:HashMap<String/* brokerAddr */, BrokerLiveInfo>变量中
                BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                    new BrokerLiveInfo(
                        System.currentTimeMillis(),
                        topicConfigWrapper.getDataVersion(),
                        channel,
                        haServerAddr));
                if (null == prevBrokerLiveInfo) {
                    log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
                }

                //对于 filterServerList 不为空的， 以 broker 地址为 key 值存入
                if (filterServerList != null) {
                    if (filterServerList.isEmpty()) {
                        this.filterServerTable.remove(brokerAddr);
                    } else {
                        this.filterServerTable.put(brokerAddr, filterServerList);
                    }
                }

                // 找到该 BrokerName 下面的主用 Broker（ BrokerId=0）
                if (MixAll.MASTER_ID != brokerId) {
                    String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                    if (masterAddr != null) {
                        ////主用 Broker 地址从brokerLiveTable 中获取 BrokerLiveInfo 对象，取该对象的 HaServerAddr 值
                        BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                        if (brokerLiveInfo != null) {
                            result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                            result.setMasterAddr(masterAddr);
                        }
                    }
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("registerBroker Exception", e);
        }

        return result;
    }

```

# 4.根据Topic获取Broker信息和Topic配置信息  #

接收到GET_ROUTEINTO_BY_TOPIC请求之后，间接调用了 RouteInfoManager.pickupTopicRouteData 方法来获取Broker和topic信息。
 
1、获取 topic 配置信息，根据 topic 从 RouteInfoManager.topicQueueTable变量中获取 List队列， 赋值给返回对象 TopicRouteData 的QueueDatas 变量。 表示该 topic 对应的所有 topic 配置信息以及每个配置所属的 BrokerName。 

2、 从上一步获取到的 List队列中获取 BrokerName 集合，该集合是去重之后的 BrokerName 集合，然后以该 BrokerName 集合的每个 BrokerName从 RouteInfoManager.brokerAddrTable 变量中获取 BrokerData 对象，将所有获取到的 BrokerData 对象集合赋值给返回对象 TopicRouteData 的 BrokerDatas集合变量。表示该 topic 是由哪些 Broker 提供的服务，以及每个 Broker 的名字、BrokerId、 IP 地址。 

3、然后以“ ORDER_TOPIC_CONFIG”和请求消息中的 topic 值为参数从NamesrvController.kvConfigManager.configTable: HashMap变量中获取orderTopiconf 值（即 broker 的顺序），并赋值给TopicRouteData.orderTopicConf 变量；该 orderTopiconf 的参数格式为：以“ ;”解析成数组，数组的每个成员是以“ :”分隔的，构成数据 “ brokerName:queueNum”；最后将 TopicRouteData 对象返回给调用者.

```
 public TopicRouteData pickupTopicRouteData(final String topic) {
        TopicRouteData topicRouteData = new TopicRouteData();
        boolean foundQueueData = false;
        boolean foundBrokerData = false;
        Set<String> brokerNameSet = new HashSet<String>();
        List<BrokerData> brokerDataList = new LinkedList<BrokerData>();
        topicRouteData.setBrokerDatas(brokerDataList);

        HashMap<String, List<String>> filterServerMap = new HashMap<String, List<String>>();
        topicRouteData.setFilterServerTable(filterServerMap);

        try {
            try {
                this.lock.readLock().lockInterruptibly();
                // 获取 topic 配置信息,表示该 topic 对应的所有 topic 配置信息以及每个配置所属的 BrokerName
                List<QueueData> queueDataList = this.topicQueueTable.get(topic);
                if (queueDataList != null) {
                    topicRouteData.setQueueDatas(queueDataList);
                    foundQueueData = true;

                    //从上一步获取到的 List队列中获取 BrokerName 集合，该集合是去重之后的 BrokerName 集合
                    Iterator<QueueData> it = queueDataList.iterator();
                    while (it.hasNext()) {
                        QueueData qd = it.next();
                        brokerNameSet.add(qd.getBrokerName());
                    }

                    //然后以该 BrokerName 集合的每个 BrokerName从 RouteInfoManager.brokerAddrTable
                    //变量中获取 BrokerData 对象，将所有获取到的 BrokerData 对象集合赋值给返回对象 TopicRouteData 的 BrokerDatas集合变量
                    for (String brokerName : brokerNameSet) {
                        BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                        if (null != brokerData) {
                            BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData
                                .getBrokerAddrs().clone());
                            brokerDataList.add(brokerDataClone);
                            foundBrokerData = true;
                            //然后以“ ORDER_TOPIC_CONFIG”和请求消息中的 topic 值为参数从NamesrvController.kvConfigManager.configTable: HashMap变量中获取orderTopiconf 值（即 broker 的顺序），
                            // 并赋值给TopicRouteData.orderTopicConf 变量；该 orderTopiconf 的参数格式为：以“ ;”解析成数组，数组的每个成员是以“ :”分隔的，
                            // 构成数据 “ brokerName:queueNum”；最后将 TopicRouteData 对象返回给调用者
                            for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
                                List<String> filterServerList = this.filterServerTable.get(brokerAddr);
                                filterServerMap.put(brokerAddr, filterServerList);
                            }
                        }
                    }
                }
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (Exception e) {
            log.error("pickupTopicRouteData Exception", e);
        }

        log.debug("pickupTopicRouteData {} {}", topic, topicRouteData);

        if (foundBrokerData && foundQueueData) {
            return topicRouteData;
        }

        return null;
    }
```

# 5.推荐 #
【1】[RocketMQ源码深度解析二之Name Server篇](https://blog.csdn.net/KilluaZoldyck/article/details/76828263)