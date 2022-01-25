---
title: cephfs部署，创建和删除
date: 2019-01-05 13:34:58
tags: cephfs
---

# cephfs部署
——————————————————————————————————————————————————————————————————————————
- 1.ceph-deploy安装
```
yum install ceph-deploy
```
- 2.ceph mds服务安装
进入到/etc/ceph目录
```
ceph-deploy gatherkeys {hostname}
ceph-deploy mds create {hostmane}:{服务名字}
```
注意可以在一个节点上面启动多个mds服务，一般推荐一个机器上面跑一个mds服务

# cephfs创建
—————————————————————————————————————————————————————————————————————————
- 1.创建cephfs需要使用的池
创建数据池
```
[root@con01 ceph(keystone_admin)]# ceph osd pool create gkk_data 20 20
pool 'gkk_data' created
```
创建元数据池
```
[root@con01 ceph(keystone_admin)]# ceph osd pool create gkk_metadata 20 20
pool 'gkk_metadata' created
```
创建cephfs文件系统
```
[root@con01 ceph(keystone_admin)]# ceph fs new gkk gkk_metadata gkk_data
new fs with metadata pool 7 and data pool 8
```
- 2.开启多文件系统模式
默认情况ceph集群只允许创建一套cephfs文件系统，但是可以打开一个flag就可以开始多文件系统模式
```
[root@con01 ceph(keystone_admin)]# ceph fs flag set enable_multiple true --yes-i-really-mean-it
```
打开flag之后创建多个cephfs文件系统如下所示
```
[root@con01 ceph(keystone_admin)]# ceph -s
    cluster 8bb2dc2f-e7f0-400e-952b-842d20e3cc1e
     health HEALTH_OK
     monmap e1: 3 mons at {con01=10.85.46.87:6789/0,con02=10.85.46.93:6789/0,con03=10.85.46.97:6789/0}
            election epoch 12, quorum 0,1,2 con01,con02,con03
      fsmap e68: ceph-1/1/1 up gkk-1/1/1 up {[ceph:0]=con03=up:active,[gkk:0]=con01=up:active}, 1 up:standby
     osdmap e255: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v1934353: 4240 pgs, 9 pools, 182 GB data, 104 kobjects
            182 GB used, 3607 GB / 3993 GB avail
                4240 active+clean
  client io 0 B/s rd, 5623 B/s wr, 0 op/s rd, 0 op/s wr
```
可以看到笔者的集群中有三个mds，有两个cephfs文件系统，名字分别叫ceph和gkk，其中他们分别占用一台mds用来管理元数据服务，分别为con03和con01
- 3.配置一个文件系统使用多个mds服务，实现使用集群来管理文件系统的元数据
```
ceph fs set ceph allow_multimds true --yes-i-really-mean-it
ceph fs set ceph max_mds 3
```
此时集群如下：
```
cluster 8bb2dc2f-e7f0-400e-952b-842d20e3cc1e
 health HEALTH_OK
 monmap e1: 3 mons at {con01=10.85.46.87:6789/0,con02=10.85.46.93:6789/0,con03=10.85.46.97:6789/0}
        election epoch 12, quorum 0,1,2 con01,con02,con03
  fsmap e74: ceph-2/2/2 up gkk-1/1/1 up {[ceph:0]=con03=up:active,[ceph:1]=con02=up:active,[gkk:0]=con01=up:active}
 osdmap e255: 3 osds: 3 up, 3 in
        flags sortbitwise,require_jewel_osds
  pgmap v1934434: 4240 pgs, 9 pools, 182 GB data, 104 kobjects
        182 GB used, 3607 GB / 3993 GB avail
            4240 active+clean
client io 10459 B/s rd, 2324 B/s wr, 12 op/s rd, 0 op/s wr
```
可以看到集群没有standby的元数据服务了

# cephfs删除
—————————————————————————————————————————————————————————————————————————
- 1.stop 所有的mds服务
```
systemctl stop ceph-mds@{hostname}
```
- 2.删除文件系统
```
ceph fs rm ceph --yes-i-really-mean-it
```
此时看一下集群状态
```
[root@con01 ceph(keystone_admin)]# ceph -s
    cluster 8bb2dc2f-e7f0-400e-952b-842d20e3cc1e
     health HEALTH_OK
     monmap e1: 3 mons at {con01=10.85.46.87:6789/0,con02=10.85.46.93:6789/0,con03=10.85.46.97:6789/0}
            election epoch 12, quorum 0,1,2 con01,con02,con03
     osdmap e264: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v1934517: 4240 pgs, 9 pools, 182 GB data, 104 kobjects
            182 GB used, 3607 GB / 3993 GB avail
                4240 active+clean
  client io 0 B/s rd, 9829 B/s wr, 0 op/s rd, 1 op/s wr
```
之前fsmap已经没有了
