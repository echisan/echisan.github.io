---
title: mysql-体系结构及存储引擎
date: 2019-09-24 14:12:45
tags:
  - mysql
  - innodb
---

## mysql主要由以下几个部分组成
- 连接池组件
- 管理服务和工具组件
- sql接口组件
- 查询分析器组件
- 优化器组件
- 缓冲（cache）组件
- 插件式存储引擎
- 物理文件

## 后台线程
mysql是多线程模型，主要有4类型的线程
- master thread（主要负责缓冲池中的数据异步刷新到磁盘）
- io thread （因采用aio，所以该线程主要处理io请求的回调）
  	- 又分为 read io，write io
- purge thread （回收undo页）
- page cleaner thread （innodb 1.2.x引入，负责脏页刷新操作，目的是减轻master thread的压力，以及用户查询线程的阻塞）

## 内存
1. 缓冲池（为了弥补cpu跟磁盘的速度，会将磁盘读取到的数据存到缓冲池中，如果有修改操作，会先更新缓冲区的数据，然后再以一定频率刷新到磁盘，值得注意的是，并非每次页发生更新时触发，而是通过**checkpoint**的机制刷新回磁盘)
2. LRU（做了一些优化，多加了一个`midpoint`，当缓冲池不能存放新读取到的页时，会释放末尾的页，而把新的数据存到`midpoint`的位置中，`midpoint`前称为`new`列表后称为`old`列表，这样做是为了避免某些sql操作会将缓冲池中的页刷新出去）
    - 这类操作主要为索引或数据的扫描操作，这类操作通常需要访问表中的许多页，这些页通常来说只会使用一次，并不是活跃的热点数据）
    - 为了继续优化这种情况，引入了`innodb_old_blocks_time`的参数，这个表示存放的`midpoint`的数据等待多久后才会被插入到new列表中
    - 用户预估自己的活跃热点数据不止`5/8`记`63%`，用户可以配置`innodb_old_blocks_pct`来减少热点页可能被刷出的概率


![image](https://user-images.githubusercontent.com/38010908/65368121-1139ec00-dc6f-11e9-885c-18d7197bcdae.png)


通过命令`show engine innodb status`观察LRU列表及free列表的使用情况和运行状态
> 注意： `show engine innodb status`显示的不是当前状态，而是过去某个时间范围内的状态 `Per second averages calculated from the last 57 seconds`

```
mysql> show engine innodb status \G
*************************** 1. row ***************************
  Type: InnoDB
  Name:
Status:
=====================================
2019-09-21 11:30:20 0x7fe7240e4700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 57 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 22 srv_active, 0 srv_shutdown, 1543541 srv_idle
srv_master_thread log flush and writes: 1543289
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 81
OS WAIT ARRAY INFO: signal count 69
RW-shared spins 0, rounds 40, OS waits 22
RW-excl spins 0, rounds 150, OS waits 5
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 40.00 RW-shared, 150.00 RW-excl, 0.00 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 2900
Purge done for trx's n:o < 2893 undo n:o < 0 state: running but idle
History list length 13
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 422105999083360, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
486 OS file reads, 367 OS file writes, 75 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number 13028166
Log flushed up to   13028166
Pages flushed up to 13028166
Last checkpoint at  13028157
0 pending log flushes, 0 pending chkp writes
42 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 227368
Buffer pool size   8192 # 这个是页为单位，一页有16kb，所以bufferPool大小为8192*16k，即128MB
Free buffers       7680 # 当前free列表中页的数量
Database pages     512 # LRU列表中页的数量
Old database pages 209
Modified db pages  0 # 表示脏页的数量
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0 # 显示LRU列表中页移动到前端的次数，因为没改变
0.00 youngs/s, 0.00 non-youngs/s
Pages read 439, created 73, written 298
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 512, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1, Main thread ID=140630651041536, state: sleeping
Number of rows inserted 4323, updated 0, deleted 0, read 7372
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.29 sec)

```

> 不知道为什么上面的没看到`buffer pool hit rate`数据，这个表示缓冲池命中率，该值通常不该小于95%。若发生小于95%的情况，就需要观察是否由于全表扫描引起的LRU列表被污染的问题


可以通过如下命令查看缓冲池的运行状态
``` 
select pool_id,hit_rate,pages_made_young,pages_not_made_young from information_schema.INNODB_BUFFER_POOL_STATS\G

*************************** 1. row ***************************
             pool_id: 0
            hit_rate: 0
    pages_made_young: 0
pages_not_made_young: 0
1 row in set (0.00 sec)
```

### 压缩页

从这里看到`LRU len: 512, unzip_LRU len: 0`有一个`unzip_LRU`的东西，是个表示压缩页的LRU。

问题： 对于压缩页的表，每个表的压缩比率可能各不相同，可能存在有的表页大小为8kb，有的大小为2kb的情况。那么`unzip_LRU`是怎样从缓冲池中分配内存的呢？

首先在unzip_LRU列表中对不同压缩页大小的页进行分别管理，其次通过伙伴算法进行内存的分配。比如对需要从缓冲池中申请页为4kb的大小，过程如下
1. 检查4kb的unzip_LRU列表，检查是否有可用的空闲页
2. 若有，直接使用
3. 否则，检查8kb的unzip_LRU列表
4. 若能够得到空闲页，将页分成2个4kb页，存放到4kb的unzip_LRU列表
5. 若不能地到空闲页，从LRU列表中申请一个16KB的页，将页分为1个8kb，2个4kb，再分别放到对应的unzip_LRU列表中

### 重做日志缓冲

一般不需要设置的很大，每一秒钟会将日志缓冲刷新到日志文件中，一般用户只需确保每秒产生的事务在这个缓冲大小内即可。
该值可由配置参数`innodb_log_buffer_size`配置，默认为8m。
下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志中
- master thread 每一秒将重做日志刷新到日志文件中
- 每个事务提交时将重做日志缓冲刷新到重做日志文件
- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到日志文件中

### 额外的内存池

缓冲池的分配，及其管理缓冲池的一些数据结构都需要从额外内存池中申请，因此当申请了很大的innodb缓冲池的时候，也应当考虑相应地增加这个值

### Checkpoint技术

> 当前事务数据库普遍采用了write ahead log 策略，当事务提交时，先写重做日志，再修改页。但发生宕机而导致数据丢失时，通过重做日志完成数据的回复。这也是事务ACID中D（durability持久性）的要求


checkpoint（检查点）的技术目的时解决以下几个问题
- 缩短数据库的恢复时间
- 缓冲池不够用时，将脏页刷新到磁盘
- 重做日志不可用时，刷新脏页


在innodb引擎内部，有两种checkpoint
- sharp checkpoint(发生在数据库关闭时，将所有的脏页都刷新会磁盘，这是默认的工作方式)
- fuzzy checkpoint(使用的这个，只刷新部分脏页)
    - Master Thread Checkpoint (差不多以每秒或每十秒从缓冲池的脏页列表异步刷新一定比例的脏页到磁盘)
    - FLUSH_LRU_LIST Checkpoint (保证LRU中有指定数量的空闲页，后面版本在Page Cleaner线程中执行)
    - Async/Sync Flush Checkpoint(当重做日志文件不可用的情况下，需要强制将一些页刷新回磁盘，此时脏页是从脏页列表中取得，为了保证重做日志的循环使用的可用性)
        > 主要有3个参数,  写入到重做日志记为`redo_lsn`，已经刷新到磁盘最新页为`checkpoint_lsn`
        > checkpoint_age = redo_lsn - checkpoint_lsn  
        > async_water_mark = 75% * total_redo_log_file_size  
        > sync_water_mark = 90% * total_redo_log_file_size  

    - Dirty Page too much Checkpoint(脏页数量太多，导致innodb存储引擎强制进行checkpoint，为了保证缓冲池中有足够可用的页，可由参数`innodb_max_dirty_pages_pct`控制，默认的值为`75`，意思是脏页数量占75%的时候会强制执行)


## Mater Thread工作方式

MasterThread具有最高的线程优先级别，内部有多个循环，master thread会根据数据库运行的状态在如下循环中切换
- 主循环(loop)
- 后台循环(backgroup loop)
- 刷新循环(flush loop)
- 暂停循环(suspend loop)





