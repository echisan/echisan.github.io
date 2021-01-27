---
title: redis-哨兵
date: 2019-10-07 17:12:11
tags:
  - redis
---

## Sentinel哨兵

Sentinel哨兵是redis高可用的解决方案：由一个或多个sentinel实例组成的sentinel系统，可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求

## 启动并初始化Sentinel
启动一个Sentinel可以使用命令
```
$ redis-sentinel /path/to/your/sentinel.conf
```
或者
```
$ redis-server /path/to/your/sentinel.conf --sentinel
```

当一个sentinel启动时，它需要执行以下步骤：
1. 初始化服务器
2. 将普通redis服务器使用的代码替换成sentinel专用代码
3. 初始化sentinel状态
4. 根据给定的配置文件，初始化sentinel的监视主服务器列表
5. 创建连向主服务器的网络连接

初始化服务器  
sentinel本质上是一个运行在特殊模式下的redis服务器，所以启动sentinel的第一步就是初始化一个普通的redis服务器，不过sentinel不实用数据库，所以初始化不会载入rdb文件或者aof文件

## 获取主服务器信息
sentinel默认会每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器当前的信息

### 获取从服务器信息
当sentinel发现主服务器有新的从服务器出现时，sentinel除了会为这个新的从服务器创建相应的实例结构外，sentinel还会创建连接到从服务器的命令连接和订阅连接

### 向主服务器和从服务器发送信息
默认情况下，sentinel会以每两秒一次的频率，通过命令连接向所有被监视的主服务器和从服务器发送以下格式命令：
```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```
命令向`__sentinel__:hello`频道发送了一条信息  
- 其中s_开头的参数记录的是sentinel本身的信息
- m_开头的记录的是主服务器的信息





