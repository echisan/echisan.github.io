---
title: redis-数据结构和对象
date: 2019-10-05 10:42:43
tags:
  - redis
---

## 简单动态字符串SDS
Redis没有直接使用c语言传统的字符串表示，而是自己构建了一种名为简单动态字符串(simple dynamic string,SDS)的抽象类型，并将SDS用作Redis的默认字符串表示

> 除了用来保存数据库中的字符串值之外，SDS还被用作缓冲区(buffer): AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区，都是又SDS实现的。

### SDS的定义
```c
struct sdshdr {
    int len;
    int free;
    char buf[];
}
```

### SDS与C字符串的区别

C字符串获取长度是需要逐个字符遍历的，时间复杂度为`O(n)`，而SDS的一个字段len用来记录长度，获取字符串长度时间复杂度为`O(1)`

### 杜绝缓冲区溢出
### 减少修改字符串带来的内存分配次数
- 空间预分配  
如果对SDS修改后，SDS的长度小于1MB，那么程序会分配和len属性同样大小的未使用空间，这是len跟free的值相同。如果大于1MB，则程序会多分配1MB的未使用空间
- 惰性空间释放  
惰性空间释放用于优化SDS字符串缩短操作。当SDS的api需要缩短sds保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节数量记录起来，并等待使用。(其实就是单纯的不会去释放free的空间，留着说不定过年的时候加字符)

### 二进制安全

C字符串会根据空字符`\0`判断字符串是不是已经读完，可能导致某些特殊格式的二进制数据被错误的获取或写入。而sds是根据len这个属性判断的。

## 链表
```c
typedef struct listNode{
    struct listNode *prev;
    struct listNode *next;
    void *value;
}listNode;
```
用过list来持有链表，主要为了方便操作
```c
typedef struct list {
    listNode *head;
    listNode *tail;
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点复制函数
    void (*free) (void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr,void *key);
}list;
```

## 字典(似乎也比较重要)

Redis字典所用的哈希表又`dict.h/dictht`结构定义
```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    // 记录的是哈希表的大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 该值总是等于size-1
    // 这个属性和哈希值一起决定一个键
    // 应该被放到table的哪个索引上
    unsigned long sizemask;
    // 该哈希表已有的节点数量
    // 记录的是记录数，即键值对的数量
    unsigned long used;
}dictht;
```

哈希表节点
```c
typedef struct dictEntry {
    void *key;
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

### rehash

扩展和收缩哈希表的工作可以通过rehash操作完成，redis对字典的哈希表执行rehash步骤：
1. 为字典ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量(也就是ht[0].used)
  - 如果执行的是扩展操作，则ht[1]的大小为第一个大于等于ht[0].used*2的2的n次方
  - 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等他ht[0].used的2的n次方
2. 将保存在ht[0]中的所有键值对rehash到ht[1]上，rehash指的是重新计算键的哈希值和索引值，然后将键值放到ht[1]的哈希表指定位置上
3. 当ht[0]所有的键值都迁移到了ht[1]之后，释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备

### 哈希表的扩展与收缩
当以下条件中的任意一个被满足时，程序会自动开始对哈希表执行扩展操作：
1. 服务器目前没有在执行`BGSAVE`或者`BGREWRITEAOF`命令，并且哈希表的负载因子大于等于1
2. 服务器目前正在执行`BGSAVE`或者`BGREWRITEAOF`命令，并且哈希表的负载因子大于等于5

其中，哈希表的负载因子可以通过以下公式计算
```
load_factor = ht[0].used / ht[0].size
```

当哈希表的负载因子小于0.1时，程序会自动对哈希表执行收缩操作

## 渐进式rehash

渐进式rehash的详细步骤
1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始
3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一
4. 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash到ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成 

> 在渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，字典的delete,find,update操作会在两个哈希表上进行。例如要查找一个键的话，程序会现在ht[0]里面进行查找，没找到再去ht[1]里面找。另外，新添加的字典的键值对一律被保存到ht[1]里面，保证了ht[0]的键值对数量只减不增



## *跳跃表

> 跳跃表(skiplist)是一种有序数据结构，通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的

跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找，还可以通过顺序性操作来批量处理节点

Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中的元素成员是比较长的字符串时，redis就会使用跳跃表作为有序集合键的底层实现

**去看看跳跃表**


## 整数集合


