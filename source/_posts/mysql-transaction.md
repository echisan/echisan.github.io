---
title: mysql-事务
date: 2019-09-30 11:44:35
tags:
  - mysql
  - innodb
---

## ACID
innodb事务完全符合ACID特性
- 原子性(atomicity)
- 一致性(consistency)
- 隔离性(isolation)
- 持久性(durability)

### 分类

从事务的理论角度来说，可以把事务分为以下几种类型
- 扁平事务(Flat Transactions)
- 带有保存点的扁平事务(Flat Transactions with Savepoints)
- 链事务(Chained Transactions)
- 嵌套事务(Nested Transactions)
- 分布式事务(Distributed Transactions)

## 事务的实现

原子性，一致性，持久性通过数据库的redo log和undo log来完成。redo log称为重做日志，用来保证事务的原子性和持久性。undo log用来保证事务的一致性。

### redo

## 事务隔离级别

- READ UNCOMMITTED
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE

## 分布式事务




