---
title: k8s-pod
date: 2019-08-16 14:41:37
tags:
  - k8s
---

## init容器
- 在创建一个pod的时候最开始创建的容器，如果有多个initContainer的话，按照定义的initContainer顺序创建，如果第一init容器创建失败的话，则会按照定义好的`restartPolicy`策略判断是否进行重启该pod，而且第二个initContainer是不会进行创建操作的。
- initContainer可以获得比appContainer更多的权限

## pause容器
kubernetes中的pause容器主要为每个业务容器提供以下功能：

- 在pod中担任Linux命名空间共享的基础；
- 启用pid命名空间，开启init进程。

例子：
在节点上允许一个pause容器
```
docker run -d --name pause -p 8880:80 jimmysong/pause-amd64:3.0
```

然后再运行一个nginx,nginx将为localhost:2368创建一个代理。
```
$ cat <<EOF >> nginx.conff
error_log stderr;
events { worker_connections  1024; }
http {
    access_log /dev/stdout combined;
    server {
        listen 80 default_server;
        server_name example.com www.example.com;
        location / {
            proxy_pass http://127.0.0.1:2368;
        }
    }
}
EOF
$ docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
```
然后再为ghost创建一个应用容器，这是一款博客软件。
```
$ docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
```

现在访问http://localhost:8880/就可以看到ghost博客的界面了。

因为pause`在pod中担任Linux命名空间共享的基础`，而nginx,ghost在启动时使用了`--ipc=container:pause --pid=container:pause`参数，所以nginx,ghost，pause共享一个命名空间，所以在宿主机访问http://localhost:8880/就会映射到80端口上，而nginx又监听着80端口，然后再proxy到2368端口上(ghost默认端口是2368)

## 安全策略
Pod 安全策略 是集群级别的资源，它能够控制 Pod 运行的行为，以及它具有访问什么的能力

## 生命周期

### pod phase(相位)
> Pod 相位的数量和含义是严格指定的。除了本文档中列举的状态外，不应该再假定 Pod 有其他的 phase 值。

下面是phase可能的值
- pending (挂起) pod已被k8s系统接受，但容器镜像尚未创建
- running (运行中) pod已被绑定到节点上且至少一个容器正在运行
- succeeded (成功) pod中的所有容器都被终止了，并且不会再重启
- failed (失败) Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- unknown (未知) 因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

### 容器探针

探针由kubelet对容器执行定期诊断，kubelet调用由容器实现的handler，有三种类型的处理程序

- ExecAction 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- TCPSocketAction 对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。   
  应该是这个
  ```
  livenessProbe:
    tcpSocket:
      port: 10086
    initialDelaySeconds: 10
    periodSeconds: 10
    failureThreshold: 60
    successThreshold: 1
    timeoutSeconds: 5
  ```
- HTTPGetAction 对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态码大于等于200 且小于 400，则诊断被认为是成功的。   
  应该是这个  
  ```
  readinessProbe:
    httpGet:
      port: 10086
      path: /actuator/health
      scheme: HTTP
    initialDelaySeconds: 10
    periodSeconds: 10
    failureThreshold: 72
    successThreshold: 1
    timeoutSeconds: 5
  ```

**该什么时候使用存活（liveness）和就绪（readiness）探针?**   
- 如果容器中的进程能够在遇到问题或者不健康的情况下自行崩溃，则不一定需要存活探针；kubelet会根据`restartPolicy`自动执行正确的操作
- 如果您希望容器在探测失败时被杀死并重新启动，那么请指定一个存活探针，并指定restartPolicy 为 Always 或 OnFailure。
- 如果要仅在探测成功时才开始向pod发送流量，那就要指定就绪探针(readinessProbe)，这种情况下就绪探针可能与存活探针相同，但是spec中的就绪探针的存在意味着pod将在没有接收到任何流量的情况下启动，并且只有探针探测成功后才开始接收流量。
- 如果只想在pod被删除时能够排除请求，则不一定需要使用就绪探针了


### 重启策略

PodSpec中有一个`restartPolicy`字段，可能的值有
- Always(默认值)
- OnFailure
- Never

## hook

Pod hook（钩子）是由Kubernetes管理的kubelet发起的，当容器中的进程启动前或者容器中的进程终止之前运行，这是包含在容器的生命周期之中。可以同时为Pod中的所有容器都配置hook。

Hook的类型包括两种：

- exec：执行一段命令
- HTTP：发送HTTP请求。

```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
``` 
在容器创建之后，容器的Entrypoint执行之前，这时候Pod已经被调度到某台node上，被某个kubelet管理了，这时候kubelet会调用postStart操作，该操作跟容器的启动命令是在异步执行的，也就是说在postStart操作执行完成之前，kubelet会锁住容器，不让应用程序的进程启动，只有在 postStart操作完成之后容器的状态才会被设置成为RUNNING。

如果postStart或者preStop hook失败，将会终止容器。


## PreSet

> Preset 就是预设，有时候想要让一批容器在启动的时候就注入一些信息，比如 secret、volume、volume mount 和环境变量，而又不想一个一个的改这些 Pod 的 template，这时候就可以用到 PodPreset 这个资源对象了。

Pod Preset 是用来在 Pod 被创建的时候向其中注入额外的运行时需求的 API 资源。

可以使用 label selector 来指定为哪些 Pod 应用 Pod Preset。




