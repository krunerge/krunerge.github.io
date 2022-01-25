---
title: 教你操作configfs
date: 2020-02-26 15:01:15
tags: python iscsi kernel
---
# 问题
我在开发ceph rbd iscsi的时候会使用configfs创建内核的相关对象，比如target，tpg等，但是创建的时候删除不了，如下所示
```
[root@openstack-con02 /sys/kernel/config/target/iscsi/iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2]# rm -rf tpgt_1
rm: cannot remove 'tpgt_1/param/OFMarkInt': Operation not permitted
rm: cannot remove 'tpgt_1/param/IFMarkInt': Operation not permitted
rm: cannot remove 'tpgt_1/param/OFMarker': Operation not permitted
rm: cannot remove 'tpgt_1/param/IFMarker': Operation not permitted
rm: cannot remove 'tpgt_1/param/ErrorRecoveryLevel': Operation not permitted
rm: cannot remove 'tpgt_1/param/DataSequenceInOrder': Operation not permitted
rm: cannot remove 'tpgt_1/param/DataPDUInOrder': Operation not permitted
rm: cannot remove 'tpgt_1/param/MaxOutstandingR2T': Operation not permitted
rm: cannot remove 'tpgt_1/param/DefaultTime2Retain': Operation not permitted
rm: cannot remove 'tpgt_1/param/DefaultTime2Wait': Operation not permitted
rm: cannot remove 'tpgt_1/param/FirstBurstLength': Operation not permitted
rm: cannot remove 'tpgt_1/param/MaxBurstLength': Operation not permitted
rm: cannot remove 'tpgt_1/param/MaxXmitDataSegmentLength': Operation not permitted
rm: cannot remove 'tpgt_1/param/MaxRecvDataSegmentLength': Operation not permitted
rm: cannot remove 'tpgt_1/param/ImmediateData': Operation not permitted
rm: cannot remove 'tpgt_1/param/InitialR2T': Operation not permitted
rm: cannot remove 'tpgt_1/param/TargetAlias': Operation not permitted
rm: cannot remove 'tpgt_1/param/MaxConnections': Operation not permitted
rm: cannot remove 'tpgt_1/param/DataDigest': Operation not permitted
rm: cannot remove 'tpgt_1/param/HeaderDigest': Operation not permitted
rm: cannot remove 'tpgt_1/param/AuthMethod': Operation not permitted
rm: cannot remove 'tpgt_1/auth/password_mutual': Operation not permitted
rm: cannot remove 'tpgt_1/auth/userid_mutual': Operation not permitted
rm: cannot remove 'tpgt_1/auth/authenticate_target': Operation not permitted
rm: cannot remove 'tpgt_1/auth/password': Operation not permitted
rm: cannot remove 'tpgt_1/auth/userid': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/tpg_enabled_sendtargets': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/fabric_prot_type': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/t10_pi': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/default_erl': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/demo_mode_discovery': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/prod_mode_write_protect': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/demo_mode_write_protect': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/cache_dynamic_acls': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/default_cmdsn_depth': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/generate_node_acls': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/netif_timeout': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/login_timeout': Operation not permitted
rm: cannot remove 'tpgt_1/attrib/authentication': Operation not permitted
```
一堆没有权限，我们知道configfs是内存文件系统，那么重启一下肯定是可以解决的，除此之外还有其他方法吗？
这里研究了一下可以通过python进行相关的操作
tpg，tpg相关多gateway，就是一个target可能有多个gateway，一个tpg对应一个gateway，里面包含了一些ip等
# tpg的删除
- 代码如下
```
from rtslib_fb.target import TPG
from rtslib_fb import root
lio_root = root.RTSRoot()
target = [tgt for tgt in lio_root.targets if tgt.wwn == 'iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2'][0]
tpg = TPG(target, 3)
tpg.delete()
```
这样就可以删除了，有一下两个注意点
- 1.iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2 是target的名字，也是内核对象的名字，也是configfs的一个目录名
```
/sys/kernel/config/target/iscsi/iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2
```
- 2.TPG构造函数中的3表示是tpg的id，学名叫tag
```
drwxr-xr-x 8 root root 0 Feb 26 14:29 tpgt_1
drwxr-xr-x 8 root root 0 Feb 26 14:39 tpgt_2
```
就是上面的1和2

# target的删除
- 代码如下
```
from rtslib_fb.target import Target
from rtslib_fb.fabric import ISCSIFabricModule
iscsi_fabric = ISCSIFabricModule()
target = Target(iscsi_fabric, wwn='iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2')
target.delete()
```
如果target下面还有tpg的话，也会自动被删除
