---
title: rebuild虚拟机之后做快照少base_image_ref信息
date: 2018-10-12 21:05:03
tags: openstack nova
---

# 问题
由于nova rebuild操作一般用于之前做了虚拟机快照镜像之后由于回滚，但是也可以用来更换镜像，比如从windows换成linux，但是rebuild操作不成功，直到P版的源码这个patch也没有合进去，patch传输门
https://review.openstack.org/#/c/305079/22/nova/compute/manager.py
，笔者手动合了这个patch到p版，心想美滋滋，不过了今天测试测出了一个bug
## 情景如下
我们一般会使用虚拟机创建快照，这个快照会作为镜像上传到glance，这个镜像我们可以称之为用户镜像，这个镜像跟一般的镜像有一个不同，会增加几个属性
```
Name	Value
image_type	snapshot
instance_uuid	<uuid of instance that was snapshotted>
base_image_ref	<uuid of original image of instance that was snapshotted>
image_location	snapshot
```
其中笔者业务需要base_image_ref这个属性，但是在虚拟机重建之后，做了镜像快照之后没有这个属性了，

# 问题分析
## 创建虚拟机的时候这个属性怎么来的
- 1.创建虚拟机源码分析
![aa](1.png)
```
def _populate_instance_for_create(self, context, instance, image,
                                      index, security_groups, instance_type,
                                      num_instances, shutdown_terminate):
    ....
    # In case we couldn't find any suitable base_image
    system_meta.setdefault('image_base_image_ref', instance.image_ref)

    system_meta['owner_user_name'] = context.user_name
    system_meta['owner_project_name'] = context.project_name
    ....
```
可以看见在创建虚拟机时在nova-api中加入的
- 2.创建虚拟机快照源码分析
创建虚拟机快照的函数是_action_create_image函数，这个是入口
![bb](2.png)
```
def _create_image(self, context, instance, name, image_type,
                  extra_properties=None):
    properties = {
        'instance_uuid': instance.uuid,
        'user_id': str(context.user_id),
        'image_type': image_type,
    }
    properties.update(extra_properties or {})

    image_meta = self._initialize_instance_snapshot_metadata(
        instance, name, properties)
    ....
```
主要是通过上面的_initialize_instance_snapshot_metadata函数从虚拟机的系统元数据中找出base_image_ref，就是上面创建虚拟机的时候加入，那rebuild虚拟机之后没有是怎么回事呢，来看一下rebuild源码分析
- 3.rebuild源码分析
rebuild虚拟机的入口函数是_action_rebuild
![cc](3.png)
rebuild函数内部有一个内部函数
```
def _reset_image_metadata():
    orig_sys_metadata = dict(instance.system_metadata)
    # Remove the old keys
    for key in list(instance.system_metadata.keys()):
        if key.startswith(utils.SM_IMAGE_PROP_PREFIX):
            del instance.system_metadata[key]

    # Add the new ones
    new_sys_metadata = utils.get_system_metadata_from_image(
        image, flavor)

    instance.system_metadata.update(new_sys_metadata)
    instance.save()
    return orig_sys_metadata
```
这个函数就是用来删除虚拟机中image前缀的，而base_image_ref正好有一个image前缀，就是创建虚拟机的时候计入的，但是加加入新的虚拟机系统元数据的时候没有加入，导致这个元数据没了，笔者的做法是在这里加上，毕竟系统元数据作为一个扩展属性，类似用户自定义的，增加一个影响不是很大，修改之后如下
```
def _reset_image_metadata(image_id):
    orig_sys_metadata = dict(instance.system_metadata)
    # Remove the old keys
    for key in list(instance.system_metadata.keys()):
        if key.startswith(utils.SM_IMAGE_PROP_PREFIX):
            del instance.system_metadata[key]

    # Add the new ones
    new_sys_metadata = utils.get_system_metadata_from_image(
        image, flavor)
    new_sys_metadata["image_base_image_ref"] = image_id  

    instance.system_metadata.update(new_sys_metadata)
    instance.save()
    return orig_sys_metadata
```

之后rebuild的虚拟机再创建快照镜像之后就没有问题了
