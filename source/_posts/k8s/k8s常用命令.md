---
title: k8s常用命令
categories:
  - k8s
tags:
  - k8s
  - 常用命令
abbrlink: c0b83bc9
date: 2025-02-06 15:12:00
---

## 基础命令
### 查看节点状态

``` shel
$ kubectl get node
```

### 查看pod状态

``` powershell
$ kubectl get po -n kube-system
# 指定查询某个pod
$ kubectl get pod -A |grep metrics-server
```

### 查看所有容器命名空间

``` powershell
$ kubectl get po --all-namespaces
```

### 删除服务

``` powershell
$ kubectl delete svc --namespace=kube-system metrics-server
```

### 删除pod

``` powershell
$ kubectl delete pod metrics-server-64c6c494dc-vw2sk -n kube-system
```

### 查看deployment信息

``` powershell
$ kubectl get deployment -n <namespace>

[root@k8s-master-01 ~]# kubectl get deployment -n kube-system
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
calico-kube-controllers   1/1     1            1           87m
coredns                   2/2     2            2           124m
metrics-server            0/1     1            0           64m
```

### 删除deployment配置（删除后pod一块删除）

``` powershell
$ kubectl delete deployment <deployment名> -n <namespace>

[root@k8s-master-01 ~]# kubectl delete deployment metrics-server -n kube-system
deployment.apps "metrics-server" deleted
```
