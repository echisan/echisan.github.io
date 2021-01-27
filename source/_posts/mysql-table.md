---
title: mysql-表
date: 2019-10-16 10:25:15
tags:
  - mysql
  - innodb
---

表

## 行记录格式

### 关于text与blob存储空间（Compressed和dynamic行记录格式）

在Innodb 1.0.x版本开始引入了新的文件格式，以前支持的`Compact`和`Redundant`格式称为`Antelope`文件格式，新的文件格式称为`Barracuda`文件格式，`Barracuda`文件格式拥有两种新的行记录格式：`Compressed`和`Dynamic`。

在新的两种记录格式对于放在BLOB中的数据采用了完全的行溢出方式。  
如下图可知，数据页中只存放20个字节的指针，实际的数据都存放在Off Page中，而之前的`Compact`和`Redundant`两种格式会存放768个前缀字节

![image](https://user-images.githubusercontent.com/38010908/66882671-ec326200-effd-11e9-805d-81f594e6fb78.png)

> `Compressed`行记录格式的另一个功能就是，存储在其中的行数据会以zlib的算法进行压缩，因此对于BLOB、TEXT、VARCHAR这类大长度类型的数据能够进行非常有效的存储

## char行结构存储
在多字节字符集类型下，char类型被明确视为变长字符类型，对于未能占满长度的字符还是填充0x20。

> 在多字节字符集的情况下，CHAR和VARCHAR的实际行存储基本是没有区别的。




