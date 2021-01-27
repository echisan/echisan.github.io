---
title: 初悉java nio channel
date: 2019-04-18 10:14:52
tags: 
  - java
  - nio
  - channel
---



## 介绍



## Channel Api

这里主要看一下`FileChannel`都有哪些api吧

| 方法                          | 描述                                       |
| ----------------------------- | ------------------------------------------ |
| int read(ByteBuffer dst)      | 从Channel中读数据到ByteBuffer              |
| long read(ByteBuffer[] dsts)  | 将Channel中的数据"分散"到ByteBuffer[]      |
| int write(ByteBuffer src)     | 将ByteBuffer中的数据写入到Channel          |
| long write(ByteBuffer[] srcs) | 将ByteBuffer[]中的数据"聚集"到Channel      |
| long position()               | 返回此通道的文件位置                       |
| FileChannel position(long p)  | 设置此通道的文件位置                       |
| long size()                   | 返回此通道的文件当前大小                   |
| FileChannel truncate(long s)  | 将此通道的文件截取为给定大小               |
| void force(boolean metaData)  | 强制将所有此通道的文件更新写入到存储设备中 |

从参数中可以看出，Channel主要与ByteBuffer接触。



## 使用demo

```java
@Test
public void test1() throws IOException {
    // 创建一个读取文件数据的通道
    // file.txt中内容为：deepDarkFantasy
    FileChannel fileChannel = FileChannel.open(Paths.get("file.txt"), StandardOpenOption.READ);
    // 创建一个非直接缓存区
    ByteBuffer buffer = ByteBuffer.allocate(512);
    // 从通道中读取数据到缓存区
    fileChannel.read(buffer);
    // 通俗讲就是切换成读模式
    buffer.flip();
    
    byte[] dst = new byte[buffer.limit()];
    // 将读取到的数据存到dst中
    buffer.get(dst, 0, buffer.limit());
    // 这里输出的也是deepDarkFantasy
    print("dst: " + new String(dst, 0, dst.length));
}
```

这是一个最基本的使用方法，~~虽然我现在不知道有什么用，跟bio的方式，倒是使用方法复杂的挺多~~

> 是不是其实这个过程是没有阻塞的？是直接读？

看下图的话，是不是上部分操作是没有那个检查操作的？直接进行阻塞那个操作（即读操作）？きになるわ

![image-20190418145559432](/images/201904/image-20190418145559432.png)

这里先跳过了，之后知道了再回来（



### 其他使用方法

好像都跟`ByteBuffer`用起来差不多？要么读要么写

不过`transferForm`好像有点意思

```java
    @Test
    public void transferTest() throws IOException {
      // 读一个文件，文件中的数据为`deepDarkFantasy`，长度为15
        FileChannel inChannel = FileChannel.open(Paths.get("src.txt"), StandardOpenOption.WRITE,StandardOpenOption.READ);
      // 打算输出的文件，是一个空的文件
        FileChannel outChannel = FileChannel.open(Paths.get("target.txt"), StandardOpenOption.WRITE);

      // 将管道`inChannel`中的数据转移到`outChannel`
        outChannel.transferFrom(inChannel,0,inChannel.size());

      // 输出15
        print(inChannel.size());
      // 输出15
        print(outChannel.size());

        outChannel.close();
        inChannel.close();

    }
```

本来以为转移的只是`通道对象`即`inChannel`中的数据，没想到是调用完该方法之后，`deepDarkFantasy`已经存在在`target.txt`文件中了。

不知道是不是被`transfer`这个单词误导了，以为转移了数据就没了，结果该数据还是存在在原来的文件中，感觉像是那种复制粘贴的操作。



### "聚集"、"分散"

> long read(ByteBuffer[] dsts)  ==> 将Channel中的数据"分散"到ByteBuffer[]
>
> long write(ByteBuffer[] srcs) ==>  将ByteBuffer[]中的数据"聚集"到Channel

有这样两个api，其实看参数就大概知道了

- 读的话就是将channel中的数据读到`ByteBuffer`数组中，但要注意的是，不是将通道中的所有数据都在数组中复制一遍，而是按照数组的顺序，依此将数据读到缓存中。

  > 本来打算看一看底层是怎么实现的，emm是sun的实现没有源码看，只有反编译看不懂了太抽象了

- 而写的话，就是把`ByteBuffer`数组中的内存依此通过channel写到文件里





## Selector

