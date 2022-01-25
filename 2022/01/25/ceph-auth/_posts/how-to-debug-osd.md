---
title: how-to-debug-osd
date: 2019-09-26 22:53:21
tags: ceph osd
---

# OSD是驻留进程怎么调试
OSD使用的驻留进程，在启动会断开进程连接
在global section下面增加这个配置
```
daemonize = false
```
可以取消驻留进程调试
