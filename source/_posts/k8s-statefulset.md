---
title: k8s-statefulSet
date: 2019-08-16 14:42:30
tags:
  - k8s
---

## 最佳实践

- StatefulSet对应的Pod的存储最好通过StorageClass来动态创建
- 为了数据安全，当删除StatefulSet中Pods或者对StatefulSet进行缩容时，Kubernetes并不会自动删除StatefulSet对应的PV，而且这些PV默认也不能被其他PVC Bound。当你确认数据无用之后再手动去删除PV的时候，数据是否删除取决于PV的ReclaimPolicy配置。ReclaimPolicy支持以下三种:
    - Retain: 意味着需要手动清理
    - Recycle, 等同于 rm -rf/thevolume/*
    - Delete, 默认值，依赖于后端的存储系统自己实现
    - > 注意：
      > - 目前只有NFS和HostPath支持Recycle;
      > - EBS,GCE PD, Azure Disk，Openstack Cinder支持Delete。

- 请小心删除StatefulSet对应的PVC，首先确保Pods已经完全Terminate，然后确定不需要Volume中的数据后，再考虑删除PV。因为删除PVC可能触发对应PV的自动删除，并根据StorageClass中的recalimPolicy配置可能造成volume中的数据丢失。
- 因为部署的是有状态应用，我们需要自己创建对应的Headless Service，注意Label要和StatefulSet中Pods的Label匹配。Kubernetes会为该Headless Service创建对应SRV Records，包含所有的后端Pods，KubeDNS会通过Round Robin算法进行选择。
- 在Kubernetes 1.8+中，你必须保证StatefulSet的spec.selector能匹配.spec.template.metadata.labels，否则会导致StatefulSet创建失败。在Kubernetes 1.8之前，StatefulSet的spec.selector如果没指定则默认会等同于.spec.template.metadata.labels。
对StatefulSet进行缩容前，你需要确认对应的Pods都是Ready的，否则即使你触发了缩容操作，Kubernetes也不会真的进行缩容操作。
