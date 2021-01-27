---
title: mysql-特性
date: 2019-09-24 14:13:30
tags:
  - mysql
  - innodb
---

## 关键特性
- 插入缓冲(Insert Buffer)
- 两次写(Double Write)
- 自适应哈希索引(Adaptive Hash Index)
- 异步IO(Async IO)
- 刷新临接页(Flush Neighbor Page)

## 插入缓冲 insert buffer

insertbuffer的数据结构为B+树

需要同时满足以下两个条件：
- 索引是辅助索引(secondary index)
- 索引不是唯一(unique)的

> mysql4.1之前每张表都有一颗insert Buffer B+树，之后全局只有一颗B+树，负责对所有的表的辅助索引进行Insert Buffer。而这颗B+数放在共享表空间中，默认也就是ibdata1中。因此通过独立表空间ibd文件回复表中数据时往往会导致CheckTable失败，因为表的辅助索引中的数据可能还在Insert Buffer中，所以通过ibd文件恢复后，还需要进行`repair table`操作来重建表上的所有辅助索引

非页节点存放的是查询的search key,构造如下

space|marker|offset
---|---|---

space 表示待插入记录所在的表空间id，innodb中每个表都有一个唯一的spaceId，占用4个字节
marker 用于兼容老版本的insert buffer，占用1个字节
offset 表示页所在的偏移量，占用4个字节


### merge insert buffer

概括地说，merge insert buffer的操作可能发生在以下几种情况
- 辅助索引页被读取到缓冲池时
- insert buffer bitmap页追踪到该辅助索引页已无可用空间时
- master thread

> innodb随机选择insert buffer B+树的一个页

## 两次写
带给Innodb存储引擎的数据库可靠性

刷新操作总会先写到doublewrite然后再写到物理页中

## 自适应哈希索引

> Innodb存储引擎会监控对表上各索引的查询，如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引(Adaptive Hash Index,AHI)

AHI是通过缓冲池的B+树构造而来的，不需要对整张表构建哈希索引。  
AHI有一个要求，即对这个页的连续访问模式必须是一样的  
例如对于(a,b)这样的联合索引页，其访问模式可以是以下情况
- where a = xxx
- where a = xxx and b = xxx

访问模式一样指的是查询的条件一样，如果交替进行上述两种查询，那么InnoDB存储引擎不会对该页构造AHI。此外，AHI还有如下要求：
- 以该模式访问了100次
- 页通过该模式访问了N次，其中N=页中记录*1/16

> 值得注意的是，哈希索引只能用来搜索等值的查询，如`select * from table where index_col='xxx'`,而对于其他查找类型，如范围查找，是不能使用哈希索引的

## 异步IO

AIO可以进行IO Merge操作  
> 在InnoDB中，read ahead方式的读取都是通过AIO完成的，脏页的刷新，即磁盘的写入操作则全部由AIO完成

## 刷新邻接页

工作原理：当刷新一个脏页时，InnoDB存储引擎会检测该页所在区(extent)的所有页，如果时脏页，如果时脏页，那么一起进行刷新。

但是需要考虑两个问题
- 是不是可能将不怎么脏的页进行了写入，而该页之后又很快变成脏页
- 固态硬盘有着较高的IOPS，是否还需要这个特性

为此，InnoDB从1.2.x版本开始提供参数`innodb_flush_neighbors`，用于控制是否启用该特性，对于传统机械硬盘建议启动该特性，对于固态硬盘有着超高IOPS性能的磁盘，建议将该值设置为0，即关闭此特性


## 启动、关闭与恢复

在关闭时，参数`innodb_fast_shutdown`影响着表的存储引擎Innodb的行为，该参数可取值为0、1、2，默认值为**1**

- 0表示在关闭mysql时，innodb需要完成所有的full purge和merge insert buffer，并且将所有的脏页刷新回磁盘，这需要一些时间，有时甚至需要几个小时来完成。如果在进行innodb升级时，必须将这个参数调为0，然后再关闭数据库
- 1是默认值，表示不需要完成上述full purge和merge insert buffer操作，但是在缓冲池中的一些数据脏页还是会刷新回磁盘
- 2表示不完成full purge和merge insert buffer操作，也不将缓冲池中的数据脏页写回磁盘，而是将日志都写入日志文件，这样不会有任何事务的丢失，但是下次mysql数据库启动时，会进行恢复操作(recovery)








