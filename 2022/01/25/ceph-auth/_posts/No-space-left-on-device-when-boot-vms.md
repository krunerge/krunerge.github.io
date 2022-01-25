---
title: 从镜像和卷启动虚拟机遇到No space left on device问题
date: 2018-09-29 19:39:21
tags: openstack nova cinder
---

# 问题
- 使用windows的qcow2镜像从镜像和卷启动虚拟机的时候，出现No space left on device问题，由于公司环境的机器分配下来之后，根目录都是20G，so small

# 解决办法
分别对镜像启动和卷启动两种情况进行解决：
## 镜像启动虚拟机
- 原因分析
从镜像启动虚拟机按理来说在glance配置开启了show_image_direct_url配置之后应该可以秒级启动虚拟机，即对glance池里面镜像的rbd做快照snap，然后clone到vms池作为虚拟机的系统盘，但是这里有一个前提，ceph只支持raw格式的虚拟机镜像，如果是qcow2格式的镜像需要先从ceph的images池把镜像下载到本地目录（/var/lib/nova/instances/\_base），然后使用qemu镜像镜像格式的转化，而我使用的镜像正好的qcow2格式的，而且是windows，这个转化成raw格式的镜像需要50G，已经超过了根目录的大小，所以会报上面的No space left on device的错误。
- cephfs解决
ceph也提高文件系统功能，我们可以把这个cephfs挂载各个计算节点的/var/lib/nova/instances/\_base目录，还能达到共享存储的效果，不过一个问题，ceph mon的level数据存储默认是在根目录，而我们的根目录只有20G，所以往往会把ceph mon撑爆（我们测试环境一般计算节点也是存储节点），mon挂了之后会导致ceph的命令都不能用，由于我们把cephfs挂载了本地目录，会导致df命令不能使用，出现一脸懵逼，发生了什么
- 软连接数据盘
幸亏公司的机器还是有其他数据盘的，数据盘的容量还是慷慨的，我们可以在数据建个软连接过去
```
# mkdir /data/base
# ln -s /data/base /var/lib/nova/instances/_base
```
## 卷启动虚拟机
- 从卷启动类型也需要先下载到本地进行转化，不过卷启动会先创建一个空的rbd卷在volumes池，然后把转化后的本地镜像通过rbd import命令导入到rbd卷中，这里的本地转化发生在/var/lib/cinder/conversion目录下，也是在根目录，解决方法跟上面类似

我用的是软连接，ok问题解决
