---
title: jmeter入门使用
date: 2019-04-23 18:07:47
tags:
  - java
  - jmeter
  - 压力测试
---



## jmeter

### 安装&运行

安装：

下载地址：https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-5.1.1.zip

运行：

由于已经有java环境了，所以不用安装java以及配置环境了，可以直接下一步操作了

找到bin目录下

```
sh jmeter
```

~~优化(可干可不干)~~

也就是把jmeter添加到环境里 不用找到bin目录而已，麻烦，不写了

## 开始

运行之后界面是这样的，这个可以把语言设置成中文的

![image](https://user-images.githubusercontent.com/38010908/67914940-8da7df00-fbcc-11e9-9aa1-5f475fd25d6a.png)
### 添加第一个测试

往`test plan`添加一个`Thread Group`

![image](https://user-images.githubusercontent.com/38010908/67914959-98fb0a80-fbcc-11e9-96bb-302636d904d0.png)


### 修改ThreadGroup

一个线程组是负责该次测试的并发控制，主要用到的大概是如下几个参数（我猜的

- Number of Threads 设置线程数（用户数）
- Loop Count 每条线程访问次数
- Duration 持续时间（也就是这些这次测试所持续的时间（我猜的

![image](https://user-images.githubusercontent.com/38010908/67915123-0e66db00-fbcd-11e9-914a-10bc863a9b7e.png)

### 添加Http请求测试（为例）

这里选择`HttpRequest`为例，后面可能会选择下面的mqtt进行测试（mqtt的需要装插件，后面再说）

![image](https://user-images.githubusercontent.com/38010908/67915157-2179ab00-fbcd-11e9-96b7-56d3495b42a4.png)



这里填写api接口信息，填写应用层协议路径啥的，跟postman差不多

![image](https://user-images.githubusercontent.com/38010908/67915195-38b89880-fbcd-11e9-8d5b-346505d9accd.png)


请求头需要再这里添加，需要注意的是电中`ThreadGroup`那个选项卡添加

![image](https://user-images.githubusercontent.com/38010908/67915213-453cf100-fbcd-11e9-97db-b3ccf24227d4.png)

添加请求头就行了，要加什么就点击下面的`add`按钮添加就可以了

![image](https://user-images.githubusercontent.com/38010908/67915234-54bc3a00-fbcd-11e9-9115-666ce2704d7e.png)

添加完这两点基本信息就填完了。



#### 请求参数从文件中读取或随机参数

**如果没有这需求可以先跳过**

按照下面的图添加一个`CVS Data Set Config`

![image](https://user-images.githubusercontent.com/38010908/67915249-63a2ec80-fbcd-11e9-8d85-9b03c0cab1d8.png)

> 说一下这个csv数据怎么来 ，如果知道的直接跳过了

1. 你可以自己vim 写一个，数据结构大概这样，各个字段用英文字母的逗号分开

   ```
   id,username,phone
   1,abc,12345
   2,efg,67890
   ```

2. 也可以在excel中写好，然后另存为csv类型，保存就完事了（大概下图这个样子

![image](https://user-images.githubusercontent.com/38010908/67915276-73223580-fbcd-11e9-868e-650b09104ebc.png)

之后即可配置jmeter中的csv模块

![image](https://user-images.githubusercontent.com/38010908/67915293-82a17e80-fbcd-11e9-92c2-a45eb2999a91.png)

> **这里要注意一下** 如果你的csv文件第一行有那些字段名的话，需要将 `ignore first line`设置为false，不然会取不到值的

到这里为止已经将csv的数据添加好了，使用csv中的数据只要像个大模版语言那样，在你需要的地方使用`${key}`的方式取出来就可以了，注意的是需要先在上图中的`Variable Names`字段填好你csv文件中的所有字段

比如在请求头管理的页面中使用

![image](https://user-images.githubusercontent.com/38010908/67915302-8fbe6d80-fbcd-11e9-936d-32f997deafd4.png)

比如在请求中的requestbody中使用

![image](https://user-images.githubusercontent.com/38010908/67915441-f8a5e580-fbcd-11e9-8e87-4a84dc972f4d.png)

**动态数据**

> 网上看来的方法，有需求的可以去测试看看
>
> 取参数也可以通过函数 动态生成数据，如 ${__RandomString(8,324YFHDDN0098432U2J32EWWDDYEHD,)}指在后面的字符中随机取8个数字。

### 添加监听器

上述操作只是将测试的api数据准备好而已，但是我们需要知道测试后的结果反馈，在jmeter中这个功能在listener中实现的，所以需要添加我们需要的监听器，常用的监听器为

- Aggregate Report （聚合报告）

  可以查看这次场景中一共完成了多少个请求，平均时间、最大/小响应时间、吞吐量等数据

- View Tree Result （结果树）

  可以查看每个请求的状态、放回结果等等



添加操作具体如下，主要也还是选中`Thread Group`那个选项，添加找到Listener，选择ViewResultsTree以及AggregateReport（注意：这是两步操作）

![image](https://user-images.githubusercontent.com/38010908/67915481-1bd09500-fbce-11e9-9906-195b91610ff3.png)



#### ViewResultsTree 结果树

这里根据个人需要，需要把测试的结果保存到某个文件上，但一般都需要的了。至于为什么后面说

![image](https://user-images.githubusercontent.com/38010908/67915953-3bb48880-fbcf-11e9-8db1-b0c886779a16.png)

#### AggregateReport 聚合报告

具体操作跟上述结果树操作差不多，可以选择需要输出的结果是errors还是successes的，按需选择了。

![image](https://user-images.githubusercontent.com/38010908/67916073-8afab900-fbcf-11e9-99c0-3184efad7a5d.png)

### 注意

在通过命令行启动jmeter的时候注意到有一段话，所以上述的一切操作都是为了生成那个`jmx文件`，gui上的运行操作仅做测试。

> Don't use GUI mode for load testing !, only for Test creation and Test debugging.
>
> For load testing, use CLI Mode (was NON GUI):
>
>    jmeter -n -t [jmx file] -l [results file] -e -o [Path to web report folder]
>
> & increase Java Heap to meet your test requirements:
>
>    Modify current env variable HEAP="-Xms1g -Xmx1g -XX:MaxMetaspaceSize=256m" in the jmeter batch file
>
> Check : https://jmeter.apache.org/usermanual/best-practices.html