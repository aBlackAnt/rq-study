# 一、NameServer代码结构

如下图：

![](https://github.com/aBlackAnt/rq-study/blob/master/nameServer/images/code_all.png?raw=true)

***NamesrvStartup***：NameServer启动类；

***RouteInfoManager***：路由信息的管理类，存放Broker的状态信息及Topic与Broker的关联关系；

***DefaultRequestProcessor***：请求的处理类，封装了对Broker发来的各种请求的响应；

***NamesrvController***：NameServer控制类，管控NameServer的启动、初始化、停止等生命周期；

***KVConfigManager***：NameServer配置信息管理类，加载NamesrvConfig中配置的配置文件到内存，此类一个亮点就是使用轻量级的非线程安全容器，再结合读写锁对资源读写进行保护，尽最大程度提高线程的并发度。

# 二、源码解析（功能）

## 2.1 NamesrvStartup

NamesrvStartup主要完成两个功能：

- 解析命令行参数；
- 初始化NamesrvController。

### 2.1.1 解析命令行参数

```java
Options options = ServerUtil.buildCommandlineOptions(new Options());
        commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
        if (null == commandLine) {
            System.exit(-1);
            return null;
        }

        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        nettyServerConfig.setListenPort(9876);
        if (commandLine.hasOption('c')) {
            String file = commandLine.getOptionValue('c');
            if (file != null) {
                InputStream in = new BufferedInputStream(new FileInputStream(file));
                properties = new Properties();
                properties.load(in);
                MixAll.properties2Object(properties, namesrvConfig);
                MixAll.properties2Object(properties, nettyServerConfig);

                namesrvConfig.setConfigStorePath(file);

                System.out.printf("load config properties file OK, %s%n", file);
                in.close();
            }
        }

        if (commandLine.hasOption('p')) {
            InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
            MixAll.printObjectProperties(console, namesrvConfig);
            MixAll.printObjectProperties(console, nettyServerConfig);
            System.exit(0);
        }
```

- -c命令行参数用来指定配置文件的位置；
- -p命令行参数用来打印所有配置项的值。

### 2.1.2 初始化NamesrvController

```java
if (null == controller) {
	throw new IllegalArgumentException("NamesrvController is null");
}

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
```

- 根据解析出来的配置参数，调用controller.initialize()方法来初始化；
- 调用controller.start()让NameServer开始服务；
- 注册ShutdownHookThread，在jvm退出的之前调用controller.shutdown()来做退出前的清理工作。

## 2.2  NamesrvController

NameServer的总控逻辑在NamesrvController中。

```java
this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        this.registerProcessor();

        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);
```

```java
private void registerProcessor() {
        if (namesrvConfig.isClusterTest()) {

            this.remotingServer.registerDefaultProcessor(new ClusterTestRequestProcessor(this, namesrvConfig.getProductEnvName()),
                this.remotingExecutor);
        } else {

            this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
        }
    }
```

- 启动一个默认是8个线程的线程池；
- 两个定时执行的线程，一个扫描失效的Broker（scanNotActiveBroker），一个打印配置信息（printAllPeriodically）。
- 启动负责通信的服务remotingServer，监听一些端口，收到Broker、Client等发过来的请求后，根据请求的命令，调用不同的processor来处理。

## 2.3 DefaultRequestProcessor

用于处理NameServer的核心业务逻辑。

```java
switch (request.getCode()) {
            case RequestCode.PUT_KV_CONFIG:
                return this.putKVConfig(ctx, request);
            case RequestCode.GET_KV_CONFIG:
                return this.getKVConfig(ctx, request);
            case RequestCode.DELETE_KV_CONFIG:
                return this.deleteKVConfig(ctx, request);
            case RequestCode.QUERY_DATA_VERSION:
                return queryBrokerTopicConfig(ctx, request);
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
            case RequestCode.UNREGISTER_BROKER:
                return this.unregisterBroker(ctx, request);
            case RequestCode.GET_ROUTEINTO_BY_TOPIC:
                return this.getRouteInfoByTopic(ctx, request);
            case RequestCode.GET_BROKER_CLUSTER_INFO:
                return this.getBrokerClusterInfo(ctx, request);
            case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
                return this.wipeWritePermOfBroker(ctx, request);
            case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
                return getAllTopicListFromNameserver(ctx, request);
            case RequestCode.DELETE_TOPIC_IN_NAMESRV:
                return deleteTopicInNamesrv(ctx, request);
            case RequestCode.GET_KVLIST_BY_NAMESPACE:
                return this.getKVListByNamespace(ctx, request);
            case RequestCode.GET_TOPICS_BY_CLUSTER:
                return this.getTopicsByCluster(ctx, request);
            case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
                return this.getSystemTopicListFromNs(ctx, request);
            case RequestCode.GET_UNIT_TOPIC_LIST:
                return this.getUnitTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
                return this.getHasUnitSubTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
                return this.getHasUnitSubUnUnitTopicList(ctx, request);
            case RequestCode.UPDATE_NAMESRV_CONFIG:
                return this.updateConfig(ctx, request);
            case RequestCode.GET_NAMESRV_CONFIG:
                return this.getConfig(ctx, request);
            default:
                break;
        }
```

- 根据RequestCode调用不同的函数来处理，从RequestCode可以了解到NameServer的主要功能，比如：REGISTER_BROKER是在集群中新加入一个Broker机器。

## 2.4 RouteInfoManager

用来保存和维护集群的各种元数据。

```java
this.topicQueueTable = new HashMap<String, List<QueueData>>(1024);
this.brokerAddrTable = new HashMap<String, BrokerData>(128);
this.clusterAddrTable = new HashMap<String, Set<String>>(32);
this.brokerLiveTable = new HashMap<String, BrokerLiveInfo>(256);
this.filterServerTable = new HashMap<String, List<String>>(256);
```

- topicQueueTable：key是topic名称，value是QueueData队列，队列的长度等于这个topic数据存储的Master Broker的个数，QueueData里存储着Broker名称，读写queue的数量、同步标识等；
- brokerAddrTable：key是BrokerName，value存储一个BrokerName对应的属性信息，包括所属的Cluster名称，一个Master和多个Slave Broker的地址信息；
- clusterAddrTable：key是clusterName，value是BrokerName组成的集合；
- brokerLiveTable：key是BrokerAddr，即一台机器，value是这台机器的实时状态，包括上次更新状态的时间戳，超时未更新则认为此Broker无效，将其从broker列表里删除；
- filterServerTable：key是Broker地址，value是和其关联的多个Filter Server的地址。

