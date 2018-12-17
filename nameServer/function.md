# NameServer功能

1、broker集群会周期地更新保存在每个name server的元数据；

2、name server集群为客户端提供最新的路由信息，客户端包括producer集群、consumer集群和command line clients。