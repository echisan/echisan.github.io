---
title: redis-集群
date: 2019-10-07 17:12:25
tags:
  - redis
---

## 集群
Redis集群是Redis提供的分布式数据库方案，集群通过分片(sharding)来进行数据共享，并提供复制和故障庄毅功能

## 节点

连接各个节点的工作可以使用`cluster meet`命令来完成
```
cluster meet <ip> <port>
```
该命令可以让node节点与ip和port指定的节点进行握手，当握手成功时，node节点就会将ip和port所指定的节点添加到node节点当前所在的集群中


### 启动节点
一个节点就是一个运行在集群模式下的redis服务器，redis服务器在启动时会根据cluster-enabled配置选项是否为yes来决定是否开启服务器的集群模式

### 集群数据结构
clusterNode结构保存了一个节点的当前状态，比如节点的创建时间、节点的名字、节点当前的配置纪元、节点的ip地址和端口号等等

```c
struct clusterNode {
    // 创建节点的时间
    mstime_t ctime;
    // 节点的名字，由40个十六进制字符组成
    char name[REDIS_CLUSTER_NAMELEN];
    // 节点标识
    // 使用各种不同的标识值记录节点的角色(比如主节点或者从节点)
    // 以及节点目前所处的状态
    int flags;
    // 节点当前的纪元，用于实现故障转移
    uint64_t configEpoch;
    // 节点的ip
    char ip[REDIS_IP_STR_LEN];
    // 保存连接节点所需的有关信息
    clusterLink *link;
}
```
clusterLink保存了连接节点所需的有关信息，比如套接字描述符，输入缓冲区和输出缓冲区：
```c
typedef struct clusterLink {
    // 连接的创建时间
    mstime_t ctime;
    // TCP套接字描述符
    int fd;
    // 输出缓冲区，保存着等待发送其他节点的消息
    sds sndbuf;
    // 输入缓冲区，保存着从其他节点接收到的消息
    sds rcvbuf;
    // 与这个连接相关联的节点，如果没有的话就为null
    struct clusterNode *node;
} clusterLink;
```
每个节点都保存着一个clusterState结构，这个结构记录了在当前节点的视角下，集群目前所处的状态，例如集群是在线还是下线，集群包含多少个节点，集群当前的配置纪元，诸如此类：
```c
typedef struct clusterState {
    // 指向当前节点的指针
    clusterNode *myself;
    // 集群当前的配置纪元，用户实现故障转移
    uint64_t currentEpoch;
    // 集群当前的状态：是在线还是下线
    int state;
    // 集群中至少处理着一个槽的节点的数量
    int size;
    // 集群节点名单(包括myself节点)
    // 字典的键为节点的名字，字典的值为节点对应的clusterNode结构
    dict *nodes;
    // ...
} clusterState;
```

![image](https://user-images.githubusercontent.com/38010908/66279710-b26aa880-e8e5-11e9-8823-b8108e95184b.png)


### cluster meet 命令实现

![image](https://user-images.githubusercontent.com/38010908/66279880-7552e600-e8e6-11e9-8b09-62dd7b99995a.png)



