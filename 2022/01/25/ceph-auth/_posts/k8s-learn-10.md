---
title: k8s learn(10)kubeadm使用
date: 2018-10-09 22:44:29
tags: k8s 张磊
---

# kubelet不能容器话
- 1.kubelet不止管理容器的生命周期，还管理网络和卷的，虽然网络比较好操作宿主机的网络，使用network=host，但是卷不能逾越到宿主机
