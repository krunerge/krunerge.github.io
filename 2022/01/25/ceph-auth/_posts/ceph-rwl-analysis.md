---
title: Intel Ceph rwl分析
date: 2019-04-13 15:17:20
tags: Ceph rwl
---

# 1. Intel Ceph rwl
Intel的rwl是对ceph的rbd进行优化，使用可持久化高速介质来缓存rbd的数据，比如SSD。rwl就是这个代码的实现，其简称就是ReplicatedWriteLog。

# 2. librbd使用
一般使用rbd都是通过调用librbd库实现，比如qemu。目前L版的代码里面有两个层次的缓存，一个是image层级的缓存，一个是object层级的缓存。Intel的rwl就是实现的image层级的缓存。
