# 一、NameServer代码结构

如下图：

![](https://github.com/aBlackAnt/rq-study/blob/master/nameServer/images/code_all.png)

***NamesrvStartup***：NameServer启动类；

***RouteInfoManager***：路由信息的管理类，存放Broker的状态信息及Topic与Broker的关联关系；

***DefaultRequestProcessor***：请求的处理类，封装了对Broker发来的各种请求的响应；

***NamesrvController***：NameServer控制类，管控NameServer的启动、初始化、停止等生命周期；

***KVConfigManager***：NameServer配置信息管理类，加载NamesrvConfig中配置的配置文件到内存，此类一个亮点就是使用轻量级的非线程安全容器，再结合读写锁对资源读写进行保护，尽最大程度提高线程的并发度。

# 二、功能实现

TODO:1、存储配置信息；2、与brokers、producers、consumers网络通信