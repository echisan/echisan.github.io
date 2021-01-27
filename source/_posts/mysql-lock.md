---
title: mysql-锁
date: 2019-09-30 11:43:36
tags:
  - mysql
  - innodb
---

## InnoDB存储引擎中的锁

锁的类型，InnoDB存储引擎实现了如下两种标准的行级锁：
- 共享锁(S Lock)，允许事务读一行数据
- 排它锁(X Lock)，允许事务删除或更新一行数据

> 如果一个事务T1已经获得了行r的共享锁，那么另外的事务T2可以立即获得行r的共享锁，因为读取并没还有改变行r的数据，称这种情况为锁兼容(Lock Compatible)，若有其他的事务T3想获得行r的排他锁，则其必须等待事务T1、T2释放行r上的共享锁——这种情况称为锁不兼容。

-|X|s
---|---|---
X|不兼容|不兼容
S|不兼容|兼容

> 需要注意的是，S和X锁都是行锁，兼容是指同一记录(row)锁的兼容性情况

### 意向锁(Intention Lock) IX
意向锁是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度(fine granularity)上进行加锁。 如下图所示

![image](https://user-images.githubusercontent.com/38010908/65810810-fb35a980-e1e1-11e9-9896-f910d181bf44.png)

总结来说的话，就是对最细粒度的对象进行上锁的话，需要对粗粒度的对象上IX锁，也就是意向锁，但是会存在兼容性的问题

InnoDB在支持意向锁设计比较简练，其意向锁即为表级别的锁。设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型。其支持两种意向锁
- 意向共享锁(IS Lock)，事务想要获得一张表中某几行的共享锁
- 意向排它锁(IX Lock)，事务想要获得一张表中某几行的排它锁

由于InnoDB支持的是行级别的锁，因此意向锁其实不会阻塞全表扫以外的任何请求。故表级意向锁与行级锁的兼容性如下
> 我理解的应该这么说 (列)想要获得表(行)上的一个(列)锁，e.g. IS想要获得表的IS锁，所以是兼容的

-|IS|IX|S|X
---|---|---|---|---
IS|兼容|兼容|兼容|不兼容
IX|兼容|兼容|不兼容|不兼容
S|兼容|不兼容|兼容|不兼容
X|不兼容|不兼容|不兼容|不兼容

### 一致性非锁定读

一致性非锁定读是指InnoBD存储引擎通过行多版本控制的方式来读取当前执行时间数据库中行的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反地，InnoDB会去读取行地一个快照数据。

![image](https://user-images.githubusercontent.com/38010908/65813202-76a85280-e204-11e9-9f31-2c0b80c7d440.png)

> 快照数据时指该行地之前版本地数据，该实现是通过undo段来完成地。而undo用来在事务中回滚数据，因此此快照数据本身是没有额外地开销。此外，读取快照数据是不需要上锁地，因为没有事务需要对历史地数据进行修改操作。在默认设置下，这是默认的读取方式，但在不同事务隔离级别下，读取的方式不同，并不是在每个事务隔离级别下都是采用非锁定的一致性读。此外，即使都是使用非锁定的一致性读，但是对于快照数据的定义也各不相同。

**多版本并发控制**  
一个行记录可能有不止一个快照数据，一般称这种技术为行多版本技术，由此带来的并发控制，称之为多版本并发控制。

下面有个例子，用于说明在READ COMMITED 与 READ REPEATABLE不同的事务隔离级别下，读取的方式区别

如下图，COMMITED READ级别下，总是读取最新的版本，即会话A最开始读取到id为1，在会话B更新了id为3，然后commit后，A仍然会读取最新的版本，所以select id为1就不存在了，因为最新的版本id是3。
而在READ REPEATABLE下，始终会读取最初始的版本，所以读取到的id仍然为1

![image](https://user-images.githubusercontent.com/38010908/65813440-d5230000-e207-11e9-8282-c52c03d82786.png)

> 需要特别注意的是，对于READ COMMITED 的事务隔离级别而言，从数据库理论的角度看，其违反了事务ACID中的I的特性，即隔离性。

### 一致性锁定读

在默认配置下，READ COMMITED 跟 REPEATABLE READ这两种事务隔离级别下都是采用非锁定读的。但在某些情况下，用户需要显式地对数据库读取操作进行加锁保证数据逻辑地一致性。

InnoDB对于select语句支持两种一致性锁定读(locking read)操作
- `SELECT ... FOR UPDATE` 对读取地行记录加一个X锁，其他事务不能对已锁定地行加上任何锁
- `SELECT ... LOCK IN SHARE MODE` 对读取地行记录加一个S锁，其他事务可以向被锁定地行加S锁，但是如果加X锁，则会被阻塞

> 对于上述连个SELECT锁定语句时，务必加上BEGIN，START TRANSACTION或者SET AUTOCOMMIT=0


### 自增长与锁

`innodb_autoinc_lock_mode`，总共有3个有效值可供设定，0、1、2

![image](https://user-images.githubusercontent.com/38010908/65813794-42389480-e20c-11e9-81a1-4eabdc119daf.png)

### 外键和锁

对于一个外键列，如果没有显示地对这个列加索引，InnoDB会自定对其加一个索引，因为这样可以避免表锁。(我：可能因为全表扫描不会采用一致性非锁定读？而会采用表锁？只在读的情况下？)

对于外键值的插入或更新，首先需要查询父表中的记录，即SELECT父表。但是对于父表的SELECT操作，不是使用一致性非锁定读的方式，因为这样会发生数据不一致的问题，因此这是使用的时`SELECT ... LOCK IN SHARE MODE`方式，即主动对父表加一个S锁。如果这时父表上已经这样加了X锁，子表上的操作会被阻塞。

![image](https://user-images.githubusercontent.com/38010908/65813876-60eb5b00-e20d-11e9-8145-c706acc8a439.png)

两个会话都没有进行COMMIT或ROLLBACK操作，而会话B被阻塞的原因是：  
id为3的父表在A中已经加了一个X锁(因为是DELETE操作？)，然和人B中又需要对id为3的行获取一个S锁，此时INSERT操作会被阻塞。

如果此时采用的是一致性非锁定读，这是Session B就会读到父表有id为3的记录，可以插入数据库。但是如果会话A对事务提交了，父表中就不存在id为3的记录。数据在父、子表就会存在不一致的情况。


## 锁的算法

### 行锁的3种算法

- Record Lock: 单个行记录上的锁
- Gap Lock：间隙锁，锁定一个范围，但不包含记录本身
- Next-Key Lock： Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身

**Next-Key Lock** 

如果一个索引有10，11，13，20四个值，那么该索引可能被next-key locking的区间为：
- (-∞,10]
- (10,11]
- (11,13]
- (13,20]
- (20,+∞]

采用Next-Key lock的锁定技术称为 Next-Key Locking。其设计的目的是为了解决幻读(Phantom Problem)，利用这种锁定技术，锁定的不是单个值，而是一个方位，是谓词锁(predict lock)的一种改进。

若事务T1已经通过next-key lock锁定了如下范围  
`(10,11]、(11,13]`  
当插入新的记录12时，则锁定的范围会变成:  
`(10,11]、(11,12]、(12,13]`  

> 然后当查询的索引含有唯一属性时，innodb会对next-key lock进行优化，将其降级为Record lock，即仅锁住索引本身，而不是范围。

举个栗子，创建了如下表(z)

a(主键)|b(辅助索引)
---|---
1|1
3|1
5|3
7|6
10|8

当在会话A执行下面的SQL语句：
```sql
SELECT * FROM z WHERE b = 3 FOR UPDATE
```

此时sql通过索引b进行查询，因为时非唯一的辅助索引，所以采用了传统的next-key locking技术，并且由于有两个索引，其需要分别进行锁定。对于聚集索引来说，其仅对列a等于5的索引加上record lock。所以对于辅助索引，其加上了Next-Key Lock，锁定的范围是(1,3)。 **需要特别注意的是，InnoDB还会对辅助索引的下一个键值加上gap lock,即还有一个辅助索引范围为(3,6)的锁** 因此，若在新会话B中允许下面的SQL语句，都会被阻塞
```sql
SELECT * FROM z WHERE a = 5 LOCK IN SHARE MODE;
INSERT INTO z SELECT 4,2
INSERT INTO z SELECT 6,5
```

分析：  
第一个sql语句不能执行，因为A中的sql已经对聚集索引列中a=5的值加上了X锁，因此会被阻塞。  
第二句sql语句，主键插入4没有问题，但是插入的辅助索引值为2在锁定范围(1,3)之间，因此会被阻塞  
第三个sql语句，主键插入6没有问题，没有被锁定，索引5插入不在(1,3)中，但是在(3,6)中，所以依然会被锁定。

但是如果执行以下sql，是不会被阻塞的  
```sql
INSERT INTO z SELECT 8,6
INSERT INTO z SELECT 2,0
INSERT INTO z SELECT 6,7
```

从上面例子可以看到，Gap Lock为了阻止多个事务将记录插入到同一范围内，而这回导致幻读的产生。  
例如上面的例子中，会话A中已经锁定b=3的记录。若此时没有gap lock锁定(3,6)，那么用户可以插入索引b列为3的记录，这会导致会话A中的用户再次执行同样查询时会返回不同的记录，即幻读(Phantom Problem)。

在innodb存储引擎中，对于insert操作，会检查插入记录的下一条记录是否被锁定，若已经被锁定，则不允许查询。


> 对于唯一键值的锁定，Next-Key Lock降级为Record Lock仅存在于查询所有的唯一索引列。若唯一索引由多个列组成，而查询仅是查找多个唯一索引列中的其中一个，那么查询其实是range类型查询，而不是point类型查询，故innodb依然使用next-key lock进行锁定

### 解决(Phantom Problem)幻读

在默认的事务隔离级别下，即REPEATABLE READ下，innodb采用next-key locking机制来避免幻读问题。

> **Phantom Problem是指同一事务下，连续执行2次同样的SQL语句可能导致不同的结果，第二次的SQL语句可能返回之前不存在的行**

举个栗子，创建一个表t,`CREATE TABLE t (a INT PRIMARY KEY);`

a|
---|
1|
2|
5|

若此时事务T1执行如下语句  
`SELECT * FROM t WHERE a>2 FOR UPDATE;`  
注意，此时事务T1并没有进行提交操作，上述应该返回5这个结果。若此时另一个事务T2插入了4这个值(假设数据库允许这个操作)那么事务T1再次执行上述SQL，会得出结果4、5，这与第一次得到的结果不同，违反了事务的隔离性。

InnoDB采用了Next-Key Locking的算法避免了幻读。对于上述sql语句，其锁住的不是5这单个值，而是对(2,+∞)这个范围加了X锁，因此对于这个范围的插入操作时不被允许的，从而避免幻读。

### 唯一性检查

背景介绍：
> 在开发过程中，很多时候，我们有很多的需求都需要“若不存在则插入”的需求。

然后目前做法是如下面的代码，但是这里会有个并发问题，因为这不是一个原子操作，在并发情况下可能会出现多个name相同的问题，当然这个可以在数据库里面做限制，name设置称unique
```golang
user := dao.getUserByName(name)
if user != nil {
    return "该用户名已存在"
}

dao.insertUser(&User{Name:name})
```

所以不设置unique的时候在想，有没有这样一种方法，但是并没有
```sql
insert into user(name) values(name) where name != 'name'
```

然后现在知道了多一个用法，当然，需要开启一个事务
```sql
select * from user where name = 'name' lock in share mode;
```
用法与上面第一种一致，但是在执行的时候，只有一个事务会成功，别的都会抛出死锁的错误
```golang
dao.insertUser(&User{Name:name})
```

## 锁问题

> 通过锁定机制可以实现事务的隔离性要求，使得事务可以并发地工作。锁提高了并发，却会带来潜在地问题。不过因为事务隔离性地要求，锁只会带来三种问题，防止这三种情况地发生，那将不会产生异常。

### 脏读
脏数据是指未提交地数据，如果读到脏数据，即一个事务可以读到另外一个事务中为提交地数据，显然违反了数据库地隔离性。

READ UNCOMMITTED(未提交读)

> 脏读隔离看似毫无用处，但在一些比较特殊地情况下还是可以将事务地隔离级别设置为READ UNCOMMITTED。例如replication环境中地slave节点，并且该slave上的查询并不需要特别精确的返回值

### 不可重复读

READ COMMITTED(提交读)

同一个事务内两次读取数据不一样的情况。一般来说，不可重复读的问题可以接受，因为其读到的是已经提交的数据，本身不会带来很大的问题。

> 在innodb中，通过使用Next-Key Lock算法来避免不可重复读。在MySQL文档中，将不可重复的也定义成幻读。在Next-Key Lock算法下，对于索引的扫描，不仅锁住扫描到的索引，而且还锁住这些索引覆盖到的范围(gap)。因此这个范围内的插入都是不允许的，这样就避免了另外的事务在这个范围内插入数据导致的不可重复度的问题。

### 丢失更新

丢失更新是另一个锁导致的问题，就是一个事务的更新操作被另一个事务的更新操作覆盖了，从而导致数据的不一致。

避免更新丢失的做法就是将操作并行化，也就是获取一个排它锁，所以在一个事务中可以这么使用
```sql
SELECT * FROM XX WHERE id=? FOR UPDATE
```

## 阻塞

在innodb中，参数`innodb_lock_wait_timeout`用来控制等待时间(默认是50秒),`innodb_rollback_on_timeout`(静态的，不可在启动时进行修改)来设定是否在等待超时时间对进行中的事务进行回滚(默认是OFF，代表不回滚)，`innodb_lock_wait_timeout`是动态的，可以在MySQL数据库运行时进行调整
```sql
SET @@innodb_lock_wait_timeout=60;
```

## 死锁

> 死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象

除了~超时机制~ 当前数据库普遍采用wait-for graph(等待图)的方式来进行死锁检测。较之超时的解决方案，这是一种更为主动的死锁检测机制。innodb也采用这种方式。wait-for graph要求数据库保存以下两种信息：
- 锁的信息链表
- 事务等待链表

通过上述链表可以构造出一张图，而在这个图中若存在回路，则代表存在死锁，因此资源间相互发生等待。在wait-for graph中，事务为图中的节点，而在图中，事务T1指向T2边的定义为：
- 事务T1等待事务T2所占用的资源
- 事务T1最终等待T2所占用的资源，也就是事务之间在等待相同的资源，而事务T1发生在事务T2的后面

看下面一个例子

![image](https://user-images.githubusercontent.com/38010908/65825941-9cd6fc80-e2af-11e9-9ec7-b3f1fee6b16c.png)

看到Transaction Wait Lists中共有4个事务，故在wait-for graph中应有4个节点

![image](https://user-images.githubusercontent.com/38010908/65825971-f2aba480-e2af-11e9-93fa-1146a2b7d69f.png)

因为t2对row1占用了x锁，t1对row2占用了s锁，t1需要等待t2中row1的资源，因此在wait-for graph中有条边从节点t1指向了t2.  
而又因为t2需要等待t1、t4所占用了row2资源，所以存在t2到t1、t4的边。  
同样，t3需要等待t1,t4,t2所占用的tow2资源，所以有一条t3到t1,t4,t2的边。  
因此最终的wait-for graph如下图所示

![image](https://user-images.githubusercontent.com/38010908/65826006-89786100-e2b0-11e9-9a11-2c7e09565bf4.png)

通过上述例子，可以发现wait-for graph是一种较为主动的死锁检测机制，在每个事务请求锁并发生等待时都会判断是否存在回路，若存在则有死锁，通常来说innodb选择回滚undo量最小的事务

> wait-for graph的死锁检测通常采用深度优先算法实现

### 锁升级

锁升级是指锁的粒度降低，e.g 行锁升级成页锁，页锁升级成表锁

> innodb根据页进行加锁，并采用位图的方式。




