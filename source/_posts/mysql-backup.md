---
title: mysql-备份与恢复
date: 2019-09-30 11:44:44
tags:
  - mysql
  - innodb
---

## 备份与恢复概述

根据备份方法的不同可以将备份分为：
- Hot Backup(热备)
- Cold Backup(冷备)
- Warm Backup(温备)

按照备份后文件的内容，备份又可以分为
- 逻辑备份
- 裸文件备份

逻辑备份：  
指备份出的文件内容是可读的，一般是文本文件。内容一般是一条条SQL语句。一般适用于数据库的升级、迁移等工作，缺点是恢复所需要的时间比较长

裸文件备份：  
指数据库的物理文件，既可以是在数据库运行中复制，也可以是在数据库停止运行时直接的数据文件复制，这类备份的恢复时间往往较逻辑备份短很多

按照备份数据库的内容来分，备份又可以分为：
- 完全备份
- 增量备份
- 日志备份

> 对于mysqldump备份工作来说，可以通过添加--single-transaction选项获得Innodb的一致性备份。此时的备份是在一个执行时间很长的事务中完成的。

## 冷备

只需要备份Mysql的frm文件，共享表空间文件，独立表空间文件(*.ibd)，重做日志文件。另外最好可以定期备份mysql的配置文件my.cnf

## 逻辑备份

mysqldump

## 逻辑备份的恢复
```
mysql -uroot -p <test_backup.sql
```

也可以
```
source /home/mysql/test_backup.sql
```

## 二进制日志备份与恢复

二进制日志非常关键，用户可以通过它完成point-in-time的恢复工作，mysql的replication同样需要二进制日志。在默认情况下并不启用二进制日志，要使用二进制日志首先必须启用它。在配置文件中进行设置  
推荐的配置：
```ini
[mysqld]
log-bin = mysql-bin
sync_binlog = 1
innodb_support_xa = 1
```

## 热备

ibbackup是innodb官方提供的热备工具，是innodb备份的首选方式，不过是收费。但是！XtraBackup是免费了，实现了ibbackup所有的功能，并且扩展支持了真正的增量备份功能





