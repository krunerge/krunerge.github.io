---
title: ceph iscsi安装
date: 2020-02-24 10:15:40
tags: ceph iscsi rbd
---
#安装包
1. 下载包
https://pan.baidu.com/s/1piNymp6oWLNHvXKaxBszLg#list/path=%2FCeph-ISCSI
2. 安装ceph-iscsi配置文件
3. 安装rbd-target-api所在的ceph-iscsi-cli包
3.1安装依赖python-configshell，网上
3.2安装包里的python-rtslib（先卸载老版本）
3.3安装包里的ceph-iscsi-config
4. 安装ceph-iscsi-tools
4.1安装依赖python-pcp，网上
4.2安装依赖pcp-pmda-lio，网上
5.安装libtcmu
6.安装targetcli
6.1安装python-ethtool，网上
7.安装tcmu-runner
8.配置/etc/ceph/iscsi-gateway.cfg

#配置
1. iscsi-gateway的api端口可能要改，默认是5000
2. gatway数目也可以iscsi-gateway配置文件中修改minimum_gateways，默认为2
3. 创建target的时候必须要主机名和ip
4. 创建好cinder配置后端所需要的池，比如iscsi池
5. 启动rbd-target-api
6. target在iscsi-gateway配置文件可以通过ceph_pool，配置gateway.conf文件所在的池
7. target在iscsi-gateway配置文件可以通过api_ip，配置rbd-target-api所监听的ip


注意
1. 不同的driver的target名字不能一样
2. configfs的内容重启可以删除
3.计算节点要安装iscsid，并配置用户名和密码
4. 替换自己修改的/usr/bin/rbd-target-api文件
5. 可以配置ceph_config_dir设置ceph配置路径
需要修改1 settings文件
```
self.cephconf = '{}/{}.conf'.format(self.ceph_config_dir, self.cluster_name)
```
需要修改2 /usr/lib/python2.7/site-packages/gwcli/ceph.py
```
UIGroup.__init__(self, 'clusters', parent)
        self.ceph_config_dir = settings.config.ceph_config_dir
        self.default_ceph_conf = '{}/{}.conf'.format(self.ceph_config_dir, settings.config.cluster_name)
```
```
把get_cluster函数的
CephGroup.ceph_config_dir改成self.ceph_config_dir
```
上面的修改涉及到ceph-iscsi-cli和ceph-iscsi-config

6. gateway ha配置
6.1 首先在每个节点安装包，然后配置/etc/ceph/iscsi-gateway.cfg,把ip都加上
6.2 在某一个节点使用gwcli创建gateway
6.3 创建gateway之后会自动同步
6.4 在为创建gwcli之前，这个节点的gwcli命令是不可用的

# cinder的参考配置如下
```
[ceph_iscsi2]
volume_driver = cinder.volume.drivers.rbd_iscsi.rbd_iscsi.RbdISCSIDriver
volume_backend_name = ceph_iscsi2
ceph_rbd_iscsi_target_ips = 10.85.46.92:3260
ceph_rbd_iscsi_target_iqns = iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2
ceph_rbd_iscsi_auth = chap/tstack12/tstack1234567
ceph_pool = iscsi
ceph_api_protocol = http
rbd_iscsi_controller_ips = 10.85.46.92:5001
username = admin
password = admin
```
multipath配置
```
[ceph_iscsi2]
volume_driver = cinder.volume.drivers.rbd_iscsi.rbd_iscsi.RbdISCSIDriver
volume_backend_name = ceph_iscsi2
#ceph_rbd_iscsi_target_ips = 10.85.46.92:3260
ceph_rbd_iscsi_target_ips = 10.85.46.94:3260,10.85.46.96:3260
ceph_rbd_iscsi_target_iqns = iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw2
ceph_rbd_iscsi_auth = chap/tstack12/tstack1234567
ceph_pool = iscsi
ceph_api_protocol = http
rbd_iscsi_controller_ips = 10.85.46.92:5001
username = admin
password = admin
```


# 多路径配置
- nova的compute配置文件要在libvirt section下面增加
```
volume_use_multipath = True
```

- 安装multipathd
```
yum install device-mapper-multipath
mpathconf --enable
systemctl restart multipathd.service
systemctl enable multipathd.service
```
```
配置如下：
devices {
        device {
                vendor                  "Tencent"
                product                 "Tsan"
                hardware_handler        "1 alua"
                path_grouping_policy    group_by_prio
                #path_selector           "queue-length 0"
                path_selector           "round-robin 0"
                failback                immediate
                path_checker            tur
                prio                    alua
                prio_args               exclusive_pref_bit
                dev_loss_tmo            30
                fast_io_fail_tmo        5
                no_path_retry           queue
        }
}
```

# 内核模块支持
tcmu-runner需要加载内核模块target_core_user，这文件路径在
```
/lib/modules/`uname -r`/kernel/drivers/target/
```
