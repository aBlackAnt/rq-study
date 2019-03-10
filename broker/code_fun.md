# 一、broker功能

broker是消息的中转站，负责消息的接收、存储、转发。从而具备了消息堆积、消费关系解耦、消息去重、顺序消息的功能。

# 二、代码结构

![1552139238842](https://github.com/aBlackAnt/rq-study/blob/master/broker/images/1552139238842.png?raw=true)

# 三、源码解析

## 3.1 broker启动

![1552139471112](https://github.com/aBlackAnt/rq-study/blob/master/broker/images/1552139471112.png?raw=true)

- 创建BrokerController

  - 设置默认的Socket发送、接收缓冲区为128KB，设置nettyServer的端口为10911。

  - 根据args[]，解析得到`commandLine`，命令参数可以有'c'、'p'、'm'，根据参数设置`brokerConfig`、`nettyServerConfig`、`nettyClientConfig`、`messageStoreConfig`

  - 在BrokerController.initialize()方法中，

    - 加载topic配置管理、消费进度管理、订阅关系管理等json文件；

    - 加载本地消息，包括检查是否正常退出、定时消息信息、加载`commitlog`文件、加载`consumequeue`文件、加载`storeCheckpoint`存储检查点、加载`indexService`消息索引服务、尝试恢复消息数据（`consumequeue`和`commitlog`实例）

    - 初始化通信层、线程池、注册请求处理器，定义了与`producer`,`consumer`,`nameserver`的交互命令码及其处理器

    - 加载定时任务

- 执行start()
  - 启动`flushConsumeQueueService`逻辑消息队列刷盘服务，默认每秒执行一次
  - 启动`flushCommitLogService`消息刷盘服务，根据配置选择同步刷盘和异步刷盘
  - 启动`storeStatsService`运行时数据统计服务
  - Master机器启动`scheduleMessageService`定时消息服务
  - 启动`reputMessageService`从物理队列解析消息重新发送到逻辑队列
  - 启动`HAService`，负责同步双写，异步复制功能
  - 创建临时文件`abort`
  - 定时删除过期文件

## 3.2 消息接收存储

![1552198148200](https://github.com/aBlackAnt/rq-study/blob/master/broker/images/1552198148200.png?raw=true)

- broker接收到sendMessage命令，根据是否批量发送，执行相应的方法

- 在`DefaultMessageStore`类的`putMessage`方法中将消息存入queue buffer中，对应CommiLog类中byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen)

![1552208610912](https://github.com/aBlackAnt/rq-study/blob/master/broker/images/1552208610912.png?raw=true)

- 根据不同的刷盘策略(同步刷盘，异步刷盘)，将消息持久化到文件

- 为了高可用，进行主从同步（同步复制，异步复制）


![1552210211848](https://github.com/aBlackAnt/rq-study/blob/master/broker/images/1552210211848.png?raw=true)

- 该实现类是一个线程函数，内部通过run操作循环去`commitLog`去消息位移信息保存到`consumeQueue`当中



## 3.3 消息存储结构

![1552206950348](https://github.com/aBlackAnt/rq-study/blob/master/broker/images/1552206950348.png?raw=true)

- 消息的存储由两部分构成，分别是`CommitLog`和`ComsumeQueue`，前者是真正的消息物理存储文件，后者是消息的逻辑队列，存储的是指向物理存储的地址，每个Topic下的每个`MessageQueue`都对应一个`ConsumeQueue`文件。
- 向`CommitLog`顺序写，随机读，充分利用操作系统的`pageCache`机制，可以批量地从磁盘读取，作为cache存到内存中，加快后续读取速度。
- 为了保证完全的顺序写，需要使用`ConsumeQueue`，其中只存偏移量信息。同时为了保证二者一致性，`CommitLog`存了`ConsumeQueues`、`MessageKey`等所有信息。

参考：

http://wuzhaoyang.me/2018/01/24/rocketmq-broker-initialize-5.html

https://www.jianshu.com/p/bc85c0695da0

http://www.iocoder.cn/RocketMQ/message-store/