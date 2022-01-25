---
title: '卷启动的虚拟机在秒起的时候可以rebuild，delete，resize操作，Why？'
date: 2019-10-04 16:14:11
tags: Openstack nova ceph
---

# 问题
————————————————————————————————————————————————————————————————————————————
最近在使用卷启动虚拟机，比如配秒起，glance接cinder，这个过程遇到一个很奇怪的问题，秒起之后卷启动的虚拟机可以delete，resize，rebuild？
为什么？镜像秒起的虚拟机就不行，在秒起之后，如果后端接的ceph，秒起其实就是snap和clone。
首先我们使用卷启动一个虚拟机，然后创建一个镜像，然后通过这个size为0的镜像在创建一个虚拟机，这里其实是从第一个虚拟机的卷的快照创建第二个虚拟机的卷，这个时候第二个虚拟机的卷就依赖于第一个虚拟机。此时如果要删除第一个虚拟机的卷应该是删除不掉的，那为什么虚拟机操作可以呢？

# 根源
————————————————————————————————————————————————————————————————————————————
看了一下代码，上面的delete，rebuild，resize都会涉及到删除操作，但是nova在删除虚拟机的时候会根据需求destroy调磁盘，代码在nova/virt/libvirt/driver.py下面
```
if destroy_disks:
    # NOTE(haomai): destroy volumes if needed
    if CONF.libvirt.images_type == 'lvm':
        self._cleanup_lvm(instance, block_device_info)
    if CONF.libvirt.images_type == 'rbd':
        self._cleanup_rbd(instance)
```
在看一下_cleanup_rbd函数
```
def _cleanup_rbd(self, instance):
    # NOTE(nic): On revert_resize, the cleanup steps for the root
    # volume are handled with an "rbd snap rollback" command,
    # and none of this is needed (and is, in fact, harmful) so
    # filter out non-ephemerals from the list
    if instance.task_state == task_states.RESIZE_REVERTING:
        filter_fn = lambda disk: (disk.startswith(instance.uuid) and
                                  disk.endswith('disk.local'))
    else:
        filter_fn = lambda disk: disk.startswith(instance.uuid)
    LibvirtDriver._get_rbd_driver().cleanup_volumes(filter_fn)
```
会走else分支，关键就在这个cleanup_volumes函数，一看就是清理卷，但是前面的rbd_driver
```
def _get_rbd_driver():
    return rbd_utils.RBDDriver(
            pool=CONF.libvirt.images_rbd_pool,
            ceph_conf=CONF.libvirt.images_rbd_ceph_conf,
            rbd_user=CONF.libvirt.rbd_user)
```
哦，其实是在nova对接的ceph池vms中去查找虚拟机id开头的rbd卷，然后删除，其实这个是对镜像启动的操作，也就是说nova没有处理卷启动情况的删除操作
