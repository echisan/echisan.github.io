---
title: k8s或docker中运行arthas出现"Unable to get pid of LinuxThreads manager thread"的问题
date: 2019-11-13 15:36:51
tags:
  - k8s
  - docker
  - arthas
  - LinuxThreads
---

## 情景模拟

现在有一个很简单的景象，把arthas复制进去，并且运行arthas-demo
```dockerfile
FROM openjdk:8-jdk-alpine
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
CMD ["java","-jar","/opt/arthas/arthas-demo.jar"]
```

然后进入容器运行arthas，会提示`Unable to get pid of LinuxThreads manager thread`这个错误
```
/opt/arthas # java -jar arthas-boot.jar 
[INFO] arthas-boot version: 3.1.3
[INFO] Found existing java process, please choose one and hit RETURN.
* [1]: 1 /opt/arthas/arthas-demo.jar
1
[INFO] arthas home: /opt/arthas
[INFO] Try to attach process 1
[ERROR] Start arthas failed, exception stack trace: 
com.sun.tools.attach.AttachNotSupportedException: Unable to get pid of LinuxThreads manager thread
	at sun.tools.attach.LinuxVirtualMachine.<init>(LinuxVirtualMachine.java:86)
	at sun.tools.attach.LinuxAttachProvider.attachVirtualMachine(LinuxAttachProvider.java:78)
	at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:250)
	at com.taobao.arthas.core.Arthas.attachAgent(Arthas.java:82)
	at com.taobao.arthas.core.Arthas.<init>(Arthas.java:28)
	at com.taobao.arthas.core.Arthas.main(Arthas.java:120)
[ERROR] attach fail, targetPid: 1
/opt/arthas # 
```

## 解决方案

google了下，发现只有在arthas的github中的issue有讨论下，那里提到解决方案有两种
1. 第一种是如下
`
docker run --init my-app
`

> 但是由于我的docker镜像是跑在k8s上的，不由得我来run，所以放弃这种方案

2. 第二种是在镜像中添加init功能
就是将[tini](https://github.com/krallin/tini)添加到镜像中

最简单的方案就是如下
```dockerfile
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["java","-jar","/opt/arthas/arthas-demo.jar"]
```

## 结果
现在查看进程会发现pid为1的进程已经变成tini了
```
/opt/arthas # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /usr/bin/tini -- java -jar /opt/arthas/arthas-demo.jar
    7 root      0:00 java -jar /opt/arthas/arthas-demo.jar
   19 root      0:00 sh
   55 root      0:00 ps -ef
/opt/arthas # 
```

再去运行arthas
```
/opt/arthas # java -jar arthas-boot.jar 
[INFO] arthas-boot version: 3.1.3
[INFO] Found existing java process, please choose one and hit RETURN.
* [1]: 7 /opt/arthas/arthas-demo.jar
1
[INFO] arthas home: /opt/arthas
[INFO] Try to attach process 7
[INFO] Attach process 7 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.                           
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'                          
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.                          
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |                         
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'                          
                                                                                

wiki      https://alibaba.github.io/arthas                                      
tutorials https://alibaba.github.io/arthas/arthas-tutorials                     
version   3.1.3                                                                 
pid       7                                                                     
time      2019-11-13 08:20:51                                                   

[arthas@7]$ 
```

![image](https://user-images.githubusercontent.com/38010908/68745453-b484fd00-0631-11ea-84b1-17e6f918ed60.png)
