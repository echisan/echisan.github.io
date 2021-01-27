---
title: 安装k8s集群
date: 2019-08-16 14:43:48
tags:
  - k8s
---

## 下载kubeadm,kubectl,kubelet

```
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

> 摘抄自这里  
> https://kubernetes.io/zh/docs/setup/independent/install-kubeadm/

## 查看镜像版本

```
kubeadm config images list
```


从国内镜像仓库下载完再修改tag

> 摘抄自这里  
> https://my.oschina.net/u/2306127/blog/3070997/print  
> https://github.com/openthings/kubernetes-tools/blob/master/kubeadm/2-images/kubernetes-pull-aliyun-1.15.2.sh

## kubeadm init
为下一步做准备 需要多加一个参数
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```


## 安装代理 
calico
https://docs.projectcalico.org/v3.8/getting-started/kubernetes/
