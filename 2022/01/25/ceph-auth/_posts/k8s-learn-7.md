---
title: k8s learn(7)深入理解容器镜像
date: 2018-10-03 20:26:34
tags: k8s 张磊
---

# Mount namespace
1. 容器进程在启动之前要做一下mount操作，要不文件系统的视角还是宿主机的
2. chroot可以改变进程的视角
3. mount namespace就是对chroot不断改良的结果
4. 容器进程一般会重新挂载/目录
5. 挂载在容器根目录上，用来为容器进程提供隔离后执行环境的文件系统，就是所谓的"容器镜像",也叫做rootfs

# rootfs
rootfs(容器镜像)一般包含如下的文件和目录
```
$ ls /
bin dev etc home lib lib64 mnt opt proc root run sbin sys tmp usr var
```

# docker核心原理
- 启动linux namespace配置
- 设置指定的Cgroups参数
- 切换进程的根目录
  - 优先使用pivot_root
  - 系统不支持pivot_root，再使用chroot

# rootfs和内核
操作系统分为rootfs和内核，rootfs就是文件，配置和目录等，可以说是操作系统的躯壳
容器只包含rootfs也就是镜像，内核是共享的宿主机的内核

# 容器的一致性
容器的一致性是指rootfs会包含可运行程序以及所需要的依赖库，所以容器在哪里运行都没问题，它有了自己所需要的所有东西

# docker镜像
docker镜像引用了layer概念，用户制作镜像每一步骤，都会生成一个层，每一个层都是一个增量的rootfs
layer需要用到联合文件系统(Union File System)

- Union File System（UnionFS）
主要功能就是将多个位置的目录联合挂载到同一个目录下面
```
mount -t aufs xxxxx
```

- 一般使用的存储驱动
```
aufs(ubuntu),没有被合并进入内核
overlay,有一些bug，比如inode会被占满
overlay2，修复了上面的bug，有望替换掉aufs，但是需要较高的内核，4.0以上，而且在docker1.12版本之后才可以使用
devicemapper, 红帽出品，但是用的不多，使用的是块设备，而上面使用的文件，但是它能支持配额的管理，其他都不行
```
- 层的分级
```
只读层(镜像层)，包含操作系统
init层，是以-init结尾的层，夹在只读层和读写层之间，这层专门用来存放/etc/hosts,/etc/resolv.conf，这层不会commit提交
可读可写层（容器层），镜像发生的增，删，改都发生在这里，这层可以commit提交
```
