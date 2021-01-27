---
title: 初悉Java nio
date: 2019-04-17 14:45:55
tags: 
  - java
  - nio
---



## 前言

### ~~抄回来的~~nio介绍

> Java NIO（New IO）是从Java 1.4版本开始引入的一个新的IO API，可以替代标准的Java IO API。
> NIO与原来的IO有同样的作用和目的，但是使用的方式完全不同，NIO支持面向缓冲区的、基于
> 通道的IO操作。NIO将以更加高效的方式进行文件的读写操作。
>
> <https://www.babywang.huangsm.xyz/2019/03/01/nio%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E5%92%8C%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8%E3%80%81/>



###本文章的目的 

主要熟悉一下java中关于nio的`Buffer`api到底是干什么的，有什么用的，搞懂一下那几个关键api方便后面进阶吧。

### 分享一下我看到的一句话

> 换句话说，BIO里用户最关心“我要读”，NIO里用户最关心”我可以读了”，在AIO模型里用户更需要关注的是“读完了”。
>
> <https://tech.meituan.com/2016/11/04/nio.html>

看到这句话简直神清气爽。

## NIO API

下面是java nio中的一些api

| 方法                   | 描述                                                 |
| ---------------------- | ---------------------------------------------------- |
| Buffer clear()         | 清空缓存区并返回对缓存区的引用                       |
| Buffer flip()          | 将缓冲区的界限设置为当前位置，并将当前位置重置为0    |
| Boolean capacity()     | 返回Buffer的capacity大小                             |
| Boolean hasRemaining() | 判断缓冲区中是否还有元素                             |
| Int limit()            | 返回Buffer的界限(limit)的位置                        |
| Buffer limit(int n)    | 设置缓冲区界限为n，并返回一个具有新limit的缓冲区对象 |
| Buffer mark()          | 对缓冲区设置标记                                     |
| int position()         | 返回缓冲区的当前位置position                         |
| Buffer position(n)     | 设置缓冲区的当前位置为n，并返回修改后的Buffer对象    |
| int remaining()        | 返回position和limit之间的元素个数                    |
| Buffer reset()         | 将位置position转到以前设置的mark所在的位置           |
| Buffer rewind()        | 将位置设为0，取消设置的mark                          |



### 创建一个非直接缓存区

```java
ByteBuffer buffer = ByteBuffer.allocate(8);
```

该代码创建了一个长度为8的字节非直接缓存区

### 写数据

如果往缓存区中写入数据，则通过调用 `buffer.put()`方法。

写入一个字节数组

```java
String str = "hello";
buffer.put(str.getBytes());
```

写入一个字节数组的底层的实现

```java
public ByteBuffer put(byte[] src, int offset, int length) {
        checkBounds(offset, length, src.length);
        if (length > remaining())
            throw new BufferOverflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            this.put(src[i]);
        return this;
}
```

此时缓存区应该是这样的，位置移动到了5（5的位置的可以写的位置）

| 下标 | 0    | 1    | 2    | 3    | 4    | 5          | 6    | 7    | 8                        |
| ---- | ---- | ---- | ---- | ---- | ---- | ---------- | ---- | ---- | ------------------------ |
| 数据 | h    | e    | l    | l    | o    |            |      |      |                          |
| 备注 |      |      |      |      |      | position=5 |      |      | Limit =8<br />capacity=8 |

### flip()

> Buffer flip()  ==> 将缓冲区的界限设置为当前位置，并将当前位置重置为0

在写入数据到缓存区之后，position的值发生了变化，如果需要将写入的数据读取出来，则需要将position改成0，而需要将限制设置在现在的位置上，因为往后就没有数据可读了，所以需要将limit设置为position（这里讲述的数字都是以上面的例子为基础）

而`flip`方法底层实现正好是做了这样一件事情，调用该方法通俗来讲就是将写模式转换为读模式。

```java
public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
}
```

调用完该方法后各个字段的数据应该是这样的：

| 下标 | 0          | 1    | 2    | 3    | 4    | 5        | 6    | 7    | 8          |
| ---- | ---------- | ---- | ---- | ---- | ---- | -------- | ---- | ---- | ---------- |
| 数据 | h          | e    | l    | l    | o    |          |      |      |            |
| 备注 | position=0 |      |      |      |      | Limit =5 |      |      | capacity=8 |


### 读数据

读数据则nio-api提供了get方法，本文以获取一段数据为例。

首先看一下该方法，该方法会将读取到的数据放到`dst`数组中，`offset`为开始读取数据的位置，`length`为需要读多少长度的数据。

```java
public ByteBuffer get(byte[] dst, int offset, int length)
```

以下为测试代码：

```java
// 创建一个可以读的最大长度的数据，用于存放读出来的数据
byte[] dst = new byte[buffer.limit()];
// 读取数据
buffer.get(dst,0,buffer.limit());
print("dst(string): "+new String(dst,0,dst.length));
// 输出的为以下内容
// dst(string): hello
```

以下代码为该方法的底层实现；可以看到是通过循环调用`get()`方法的

```java
    public ByteBuffer get(byte[] dst, int offset, int length) {
        checkBounds(offset, length, dst.length);
        if (length > remaining())
            throw new BufferUnderflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            dst[i] = get();
        return this;
    }
```

查看该`get()`方法可以看到，该方法会获取当前位置的数据，读完之后会增加位置的值，即`position++`，也就是说`get()`操作会改变位置的值。

```java
    /**
     * Relative <i>get</i> method.  Reads the byte at this buffer's
     * current position, and then increments the position.
     *
     * @return  The byte at the buffer's current position
     *
     * @throws  BufferUnderflowException
     *          If the buffer's current position is not smaller than its limit
     */
    public abstract byte get();
```



### flip、rewind、clear以及compact的区别

> Buffer flip()       ==> 将缓冲区的界限设置为当前位置，并将当前位置重置为0
>
> Buffer rewind() ==> 将位置设为0，取消设置的mark
>
> Buffer clear()    ==>  字面意思上是清除缓存数据，则实现是将位置设置成0，取消mark，将limit设置成	  capacity
>
> Buffer compact() ==> 字面意思就是压缩缓存（下文详细说）



但是查看这四个方法，会发现`flip()` `rewind()` `clear()` 这三个方法的实现几乎是一样的，唯一的区别就在于是否会对`limit`进行处理。

```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}

public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}

public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}

public ByteBuffer compact() {
    System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
    position(remaining());
    limit(capacity());
    discardMark();
    return this;
}
```

至于分别有什么用、什么使用场景目前暂时未知，但是需要清楚这几个方法的效果：

- flip：在写入数据之后调用该方法后再读取数据，只能读到`[0,limit)`之间的数据

- rewind：重置标记，但是没有修改`limit`的值，而`limit`默认的值为`capacity`，如果没有修改过`limit`的值的情况下，可以读取到缓存区中所有的数据`[0,capacity)`，但是如果`limit`的值被修改过，则只能读取到的数据只有`[0,limit)`

  > The `Buffer.rewind()` sets the `position` back to 0, so you can reread all the data in the buffer. The `limit` remains untouched, thus still marking how many elements (bytes, chars etc.) that can be read from the `Buffer`.
  >
  > <http://tutorials.jenkov.com/java-nio/buffers.html>

- clear：字面上意思是清楚缓存，表示可以从位置0开始写入数据。详情可以看以下说明，说的很清楚了。

  > Once you are done reading data out of the `Buffer` you have to make the `Buffer` ready for writing again. You can do so either by calling `clear()` or by calling `compact()`.
  >
  > If you call `clear()` the `position` is set back to 0 and the `limit` to `capacity`. In other words, the `Buffer` is cleared. The data in the `Buffer` is not cleared. Only the markers telling where you can write data into the `Buffer` are.
  >
  > If there is any unread data in the `Buffer` when you call `clear()` that data will be "forgotten", meaning you no longer have any markers telling what data has been read, and what has not been read.
  >
  > http://tutorials.jenkov.com/java-nio/buffers.html

- compact：通俗的话来说就是将未读的数据设置到缓存的头部，并将位置设置在未读数据的尾部

  > `compact()` copies all unread data to the beginning of the `Buffer`. Then it sets `position` to right after the last unread element. The `limit` property is still set to `capacity`, just like `clear()` does. Now the `Buffer` is ready for writing, but you will not overwrite the unread data.
  >
  > http://tutorials.jenkov.com/java-nio/buffers.html

#### compact

探究一下compact的实现，看看具体做的事什么操作。经过该一系列操作，会将未读的数据存放在缓存的头部，后续的写入操作不会覆盖掉未读的数据。而`clear`只是将`position`设置为0，写入操作会覆盖掉数据，两者的差别在次。

```java
public ByteBuffer compact() {
    // 1. 复制数组，将未读的内容复制到缓存的头部
    System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
    // 2. 将位置设置成复制后的数据的结尾，可以查看下方的remaining()方法的实现
    position(remaining());
    // 3. 将limit设置成capacity，即整个缓存的长度
    limit(capacity());
    // 4. 重置标记
    discardMark();
    return this;
}

public final int remaining() {
    return limit - position;
}
```



## 参考文章

<https://www.babywang.huangsm.xyz/2019/03/01/nio%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E5%92%8C%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8%E3%80%81/>

 <http://tutorials.jenkov.com/java-nio/buffers.html>

 <https://tech.meituan.com/2016/11/04/nio.html>