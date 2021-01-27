---
title: redis-事务
date: 2019-10-07 17:12:34
tags:
  - redis
---

## 事务
redis通过`multi`,`exec`,`watch`等命令来实现事务功能。

例如
```
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set "name" "practical common lisp"
QUEUED
127.0.0.1:6379> get "name"
QUEUED
127.0.0.1:6379> set "author" "peter seibel"
QUEUED
127.0.0.1:6379> get "author"
QUEUED
127.0.0.1:6379> exec
1) OK
2) "practical common lisp"
3) OK
4) "peter seibel"
```

> 像是sql里的，`start transaction`,`commit`

## 事务的实现

一个事务从开始到结束通常会经历以下三个阶段
1. 事务开始
2. 命令入队
3. 事务执行

`multi`命令标志着事务的开始  
如果客户端发送的命令为`exec`,`discard`,`watch`,`multi`命令其中一个，那么服务器立即执行这个命令，相反，发送的命令是上述4个以外的命令， 则不会立即执行这个命令，而会将这个命令放入一个事务队列里面，然后向客户端返回queue回复。

![image](https://user-images.githubusercontent.com/38010908/66283095-be119b80-e8f4-11e9-88fb-8101e7e5d34c.png)


## WATCH命令的实现
watch命令是一个乐观锁，他可以在exec命令执行之前，监视任意数量的数据库键，并在exec命令执行时，检查被监视的键是否至少有一个被修改过了，如果是的话，服务器将拒绝执行事务，并向客户端返回代表事务执行失败的空回复


