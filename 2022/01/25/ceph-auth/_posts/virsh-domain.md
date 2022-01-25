---
title: virsh-domain.c文件下命令分析
date: 2018-10-09 21:34:27
tags: libvirt
---

# libvirt
由于最近要做虚拟机在线快照，熟悉一下libvirt源码以及使用
## virsh attach-device,通过xml给虚拟机添加设备
## virsh attach-disk，增加盘
## virsh attach-interface，增加网卡
## virsh autostart，设置自动启动，当libvirtd重启的时候虚拟机重启
## virsh blkdeviotune，设置磁盘I/O节流
## virsh blkiotune,设置磁盘I/O节流
