---
title: allow other lvm in lvmgroup
date: 2018-12-15 12:47:34
tags: openstack lvm
---
# 问题描述：
笔者有一套openstack环境，没有对接ceph，使用的lvm作为虚拟机的存储，虚拟机的系统卷都存放在一个叫做vms-group的lvmgroup中，不过其下有其他人创建了一个lvm的逻辑卷，但是不是给openstack的用的，导致OpenStack hypvisor中的local_gb_free不准确，导致虚拟机创建有问题

# 原理分析
通过源码分析可以知道，nova compute会定时同步hypervisor的资源，包括存储，那按理来说，从宿主机上面获取lvmgroup的总量和使用量应该是没有问题的，那为什么会导致出现最后hypervisor上面的资源统计不正确呢？
我们来看一下代码，代码是不会说谎的
```
def update_available_resource(self, context):

    LOG.info(_LI("Auditing locally available compute resources for "
                 "node %(node)s"),
             {'node': self.nodename})
    resources = self.driver.get_available_resource(self.nodename)
    ...
```
在nova/compute/resource_tracker.py文件中的update_available_resource函数调用了get_available_resource函数，这个函数其实就是调用libvirt的接口，其实调用的是如下的命令
```
vgs --noheadings --nosuffix --separator --units b -o vg_size,vg_free vg
```
就是取出vg这个逻辑组的总量和剩余量，而使用量是这样计算的
```
{'total': int(info[0]),
  'free': int(info[1]),
  'used': int(info[0]) - int(info[1])}
```
可以看出这个使用量是包括这个lvmgroup中不是openstack创建使用的，那为什么最后的hypervisor统计不正确呢
我们继续往下看
```
def update_available_resource(self, context):
    .....
    self._update_available_resource(context, resources)
```
在update_available_resource函数中还调用了更新可用量的请求，\_update_available_resource
```
def _update_available_resource(self, context, resources):

    ....
    self._update_usage_from_instances(context, resources, instances)
    .....
```
这个函数调用\_update_usage_from_instances函数，统计虚拟机的使用量，我们好像有点眉目了，我们可以猜测一下，通过统计数据库中虚拟机存盘的使用量（系统盘）来统计openstack的使用量，这就不包含了lvmgroup中其他用途创建的逻辑卷，是不是这样，我们看一下
```
def _update_usage_from_instances(self, context, resources, instances):

        # set some initial values, reserve room for host/hypervisor:
        resources['local_gb_used'] = CONF.reserved_host_disk_mb / 1024
        resources['memory_mb_used'] = CONF.reserved_host_memory_mb
        resources['free_ram_mb'] = (resources['memory_mb'] -
                                    resources['memory_mb_used'])
        resources['free_disk_gb'] = (resources['local_gb'] -
                                     resources['local_gb_used'])
        resources['current_workload'] = 0
        resources['running_vms'] = 0

        # Reset values for extended resources
        self.ext_resources_handler.reset_resources(resources, self.driver)

        for instance in instances:
            if instance.vm_state != vm_states.DELETED:
                self._update_usage_from_instance(context, resources, instance)
```
上面关键的是
```
resources['local_gb_used'] = CONF.reserved_host_disk_mb / 1024
```
local_gb_used这个参数被重新赋值了，之前的值就是通过上面调用命令之后计算出来的也就是真实值，这里进行了重新赋值，赋予了保留值，然后通过遍历数据库中的所有虚拟机，根据它们的机型中定义的系统盘值统计使用量，就是下面这个函数
```
_update_usage_from_instance
```
最后可以使用的free_gb，就是通过命令取出来的总值，减去预留值，减去虚拟机的使用量，这就导致没有包括其他用途使用的逻辑卷

# 解决方案
通过修改小修改\_update_usage_from_instances函数来解决，笔者通过取两个使用量统计的最大值最为使用量，
一个是通过命令行获取，一个是预留值加上所有虚拟机的总使用量
```
def _update_usage_from_instances(self, context, resources, instances):
    """Calculate resource usage based on instance utilization.  This is
    different than the hypervisor's view as it will account for all
    instances assigned to the local compute host, even if they are not
    currently powered on.
    """
    self.tracked_instances.clear()

    # purge old stats and init with anything passed in by the driver
    self.stats.clear()
    self.stats.digest_stats(resources.get('stats'))

    # set some initial values, reserve room for host/hypervisor:
    resources['local_gb_used_orig'] = resources['local_gb_used']
    resources['local_gb_used'] = 0
    resources['memory_mb_used'] = CONF.reserved_host_memory_mb
    resources['free_ram_mb'] = (resources['memory_mb'] -
                                resources['memory_mb_used'])
    resources['free_disk_gb'] = (resources['local_gb'] -
                                 resources['local_gb_used'])
    resources['current_workload'] = 0
    resources['running_vms'] = 0

    # Reset values for extended resources
    self.ext_resources_handler.reset_resources(resources, self.driver)

    for instance in instances:
        if instance.vm_state != vm_states.DELETED:
            self._update_usage_from_instance(context, resources, instance)

    resources['local_gb_used'] = max(resources['local_gb_used'], resources['local_gb_used_orig']) + \
                                 CONF.reserved_host_disk_mb / 1024
    resources['free_disk_gb'] = (resources['local_gb'] -
                                 resources['local_gb_used'])
```
