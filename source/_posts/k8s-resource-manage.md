---
title: k8s集群资源管理
date: 2019-08-16 14:36:39
tags:
  - k8s
---

## Node
Node是kubernetes集群的工作节点，可以是物理机也可以是虚拟机。

### Node的状态
```
kubectl describe nodes
```

### Node的管理
禁止pod调度到该节点上
```
kubectl cordon <node>
```
---

驱逐该节点上的所有pod
```
kubectl drain <node>
```
> 该命令会删除该节点上的所有Pod（DaemonSet除外），在其他node上重新启动它们，通常该节点需要维护时使用该命令。直接使用该命令会自动调用kubectl cordon <node>命令。当该节点维护完成，启动了kubelet后，再使用kubectl uncordon <node>即可将该节点添加到kubernetes集群中。

## Namespace

区分环境

```
kubectl get ns
```

## Label

label是个键值对，对系统本身没什么含义，只是对用户才有意义

```
- labels
    - key1: value1
    - key2: value2
```

> 不要在label中使用大型、非标识的结构化数据，记录这样的数据应该用annotation。


### Label Selector

Label不是唯一的，很多object可能有相同的label。

通过label selector，客户端／用户可以指定一个object集合，通过label selector对object的集合进行操作。

Label selector有两种类型：

- equality-based ：可以使用=、==、!=操作符，可以使用逗号分隔多个表达式
- set-based ：可以使用in、notin、!操作符，另外还可以没有操作符，直接写出某个label的key，表示过滤有某个key的object而不管该key的value是何值，!表示没有该label的object

示例
```
$ kubectl get pods -l environment=production,tier=frontend
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
$ kubectl get pods -l 'environment in (production, qa)'
$ kubectl get pods -l 'environment,environment notin (frontend)'
```

在API object中设置label selector

要注意的一点是，部分object不支持set-based操作,比如`service`,`replicationcontroller`

```
selector:
    component: redis
```

而在`Job`,`Deployment`,`ReplicaSet`,`DaemonSet`这些object则支持
```
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

### Annotation

> Annotation可以将Kubernetes资源对象关联到任意的非标识性元数据。使用客户端（如工具和库）可以检索到这些元数据。


以下列出了一些可以记录在 annotation 中的对象信息：

- 声明配置层管理的字段。使用annotation关联这类字段可以用于区分以下几种配置来源：客户端或服务器设置的默认值，自动生成的字段或自动生成的 auto-scaling 和 auto-sizing 系统配置的字段。
- 创建信息、版本信息或镜像信息。例如时间戳、版本号、git分支、PR序号、镜像哈希值以及仓库地址。
- 记录日志、监控、分析或审计存储仓库的指针
- 可以用于debug的客户端（库或工具）信息，例如名称、版本和创建信息。
- 用户信息，以及工具或系统来源信息、例如来自非Kubernetes生态的相关对象的URL信息。
- 轻量级部署工具元数据，例如配置或检查点。
- 负责人的电话或联系方式，或能找到相关信息的目录条目信息，例如团队网站。


### Taint和Toleration（污点和容忍）

节点可以有一个或多个taint，对于不能容忍那些taint的pod是不会被调度到那些nodes上的。如果将toleration应用于pod上，则表示这些pod可以被调度到具有相应taint的节点上


为node设置taint
```
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node2 key2=value2:NoExecute
```

删除taint
```
kubectl taint nodes node1 key1:NoSchedule-
```

查看taint
```
kubectl describe nodes node1
```
[详细用法](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)


为pod设置toleration
```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
- key: "node.alpha.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

- value 的值可以为 NoSchedule、PreferNoSchedule 或 NoExecute。
- tolerationSeconds 是当 pod 需要被驱逐时，可以继续在 node 上运行的时间。

> 如果 pod 存在一个 effect 值为 NoExecute 的 toleration 指定了可选属性 tolerationSeconds 的值，则表示在给节点添加了上述 taint 之后，pod 还能继续在节点上运行的时间


### 垃圾收集

> Kubernetes 垃圾收集器的角色是删除指定的对象，这些对象曾经有但以后不再拥有 Owner 了。

Owner会包含一个或多个Dependent(pod)

> 有时，Kubernetes 会自动设置 ownerReference 的值。例如，当创建一个 ReplicaSet 时，Kubernetes 自动设置 ReplicaSet 中每个 Pod 的 ownerReference 字段值。在 1.6 版本，Kubernetes 会自动为一些对象设置 ownerReference 的值，这些对象是由 ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment 所创建或管理。

> 也可以通过手动设置 ownerReference 的值，来指定 Owner 和 Dependent 之间的关系。

控制垃圾收集器删除Dependent有三种策略
- background    
  在 background 级联删除 模式下，Kubernetes 会立即删除 Owner 对象，然后垃圾收集器会在后台删除这些 Dependent。
- foreground   
  删除了所有"blocking"的dependent之后再立刻删除owner
- orphan(孤儿)   
  就是只删除owner，dependent就不管了




