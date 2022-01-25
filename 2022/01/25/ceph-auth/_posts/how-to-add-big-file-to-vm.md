---
title: 快速上传大文到虚拟机
date: 2020-03-07 11:23:18
tags:
---

# 问题
openstack里面使用硬件sdn，没有namespace登录到虚拟机，怎么快速从宿主机到虚拟机传送文件数据,一同事想出了方法，加以记录

# 方法
```
没有命名空间传大文件方法，最好不要用业务虚拟机测试，新建同网段的虚拟机挂载。
1. 将大文件制作成iso, 比如mkisofs -o xxx.iso xxx/
2. 上传iso到虚拟机所在的计算节点，比如/root下
3. 找到虚拟机的instance name, nova show <server uuid> | grep instance_name, 假设为instance-00000108
4. 登陆虚拟机，lsblk查看下目前最后一个盘名，假设vdb，那么将iso挂载成下一个设备。
5. 计算节点执行， virsh attach-disk instance-00000108 /root/xxx.iso  vdc
6. 执行lsblk， 可以看到新挂载的设备，mount即可。
```

# 注意
如果出现权限问题，需要把iso文件放到tmp目录下
