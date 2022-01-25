---
title: 虚拟机删除流程分析
date: 2018-10-16 21:30:59
tags: openstack nova
---

# 虚拟机删除
虚拟机的删除笔者之前没怎么研究过，心想无非就是调用libvirt接口进行destroy，其实仔细一看还是挺复杂的，让我们来分析一下

# 代码流程解析
nova-api那边就不用看了，重点在nova-compute那端，分析如下：
代码路径nova/comnpute/manager.py
- 1.同步加锁调内部函数
![aaa](1.png)
- 2.重头戏
这里_delete_instance函数还增加了一个装饰器，用来增加一些钩子函数处理
```
@hooks.add_hook("delete_instance")
def _delete_instance(self, context, instance, bdms, quotas):
    ....
```
2.1 clear_events_for_instance
删除虚拟机的事件

----
2.2 \_shutdown_instance
主要是虚拟机关机，销毁网络设备，删除ceph系统盘，删除本地instance下目录，然后销毁虚拟机（undefine）
2.2.1 首先获取网络信息
```
compute_utils.get_nw_info_for_instance(instance)
```
2.2.2 获取磁盘信息
```
_get_instance_block_device_info
```
---
2.2.3 调用driver的destroy信息
2.2.3.1 调用_destroy
强制关闭虚拟机，相关于拔电源，调用libvirt的destroy的接口，同步操作
2.2.3.2 调用cleanup进行清理
主要做一下四件事：
```
1.删除网络设备，包括linux bridge上面的qbr口，删除linux bridge以及删除br-int上面的port
2.删除ceph vms池下面的系统盘
3.删除计算节点本地的instances目录
4.调用libvirt的undefineFlags接口，销毁虚拟机
```
----
2.2.4 调用neutron删除虚拟机网络信息
```
self._try_deallocate_network(context, instance, requested_networks)
```
2.2.4.1 删除port
2.2.4.2 删除虚拟机nova数据库中的网络信息缓存

---
2.2.5 对卷进行卸载操作
2.2.5.1 terminate_connection卸载掉宿主机与卷的联系
由于笔者使用的ceph rbd，所以这个函数是空
2.2.5.2 调用cinder的detach，主要是进行数据库操作
----
2.3 删除虚拟机缓存信息
```
instance.info_cache.delete()
```
2.4 删除设置为terminate时要删除的卷
```
self._cleanup_volumes(context, instance.uuid, bdms,
                    raise_exc=False)
```
2.5 删除数据库中虚拟机instance实例
2.6 删除bdm数据库
2.7 删除console缓存token
2.7 在host manager内存dict中删除虚拟机信息

分析完毕，还是做了很多事情的
