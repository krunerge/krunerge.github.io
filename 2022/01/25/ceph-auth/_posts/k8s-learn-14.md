---
title: k8s learn(14)为什么我们需要pod
date: 2018-10-17 22:58:42
tags: k8s 张磊
---

# 1. k8s中api对象
- 1.1.metadata表示对象，主要有label
- 1.2.spec具体api对象信息

# 2.volume属于pod，类似于磁盘属于虚拟机
# 3.empty宿主机临时目录，hostpath宿主机某个目录
# 4.kubectl exec进入容器
# 5.pod的生命周期跟基础容器相同
# 6.有两种容器
- initcontainer：运行网就退出，先于container运行
- container：initcontainer运行完在运行

# 7.容器设计模式--sidecar
- 运用网络共享：Istio
- 运用卷共享：日志收集
