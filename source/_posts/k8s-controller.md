---
title: k8s控制器
date: 2019-08-16 14:40:47
tags:
  - k8s
---

## 控制器
k8s中内建了很多controller，相当于一个状态机，用来控制pod的具体状态和行为


### Deployment

为Pod和ReplicaSet提供一个声明式定义方法，用于替代以前的ReplicationController。

典型使用场景
- 定义Deployment来创建pod和ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续deployment

kubectl操作  
扩容
```
kubectl scale deployment nginx-deployment --replicas 10
```

如果集群支持自动扩容
```
kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu=percent=80
```

更新镜像
```
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
```

回滚
```
kubectl rollout undo deployment/nginx-deployment
```

指定版本
```
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

> Deployment结构示意图   
> https://jimmysong.io/kubernetes-handbook/images/deployment-cheatsheet.png

![](https://jimmysong.io/kubernetes-handbook/images/deployment-cheatsheet.png)


查看deployment升级的历史记录
```
kubectl rollout history deployment/nginx-deployment
```
> 创建 Deployment的时候使用--record参数可以记录命令，可以很方便的查看每次 revision 的变化。

查看单个revision 的详细信息：
```
kubectl rollout history deployment/nginx-deployment --revision=2
```



