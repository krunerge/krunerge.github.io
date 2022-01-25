---
title: 修改openstack源码之后如何做rpm包
date: 2018-09-29 20:18:20
tags: openstack
---

# rpm包的需求
- 以前我们修改openstack的源码之后，如何部署openstack的呢？
```
首先使用openstack原生组件的rpm包，然后把我们修改的代码用脚本镜像文件替换，额。。。好像有点low
```
# rpm制作流程
## 首先找源码的src.rpm包
来，传输门：http://vault.centos.org/7.5.1804/cloud/Source/openstack-pike/
选择要使用的rpm包，这里我们已cinder-11.1.1为例
- 下载src.rpm包
![aa](1.png)
- 解压rpm包
```
rpm -i openstack-cinder-11.1.1-1.el7.src.rpm
```
这会生成/root/rpmbuild目录
```
# ll
总用量 4
drwxr-xr-x. 3 root root   26 9月  29 19:10 BUILD
drwxr-xr-x. 2 root root    6 9月  29 19:21 BUILDROOT
drwxr-xr-x. 3 root root   19 9月  29 19:21 RPMS
drwxr-xr-x. 2 root root 4096 9月  29 19:30 SOURCES
drwxr-xr-x. 2 root root   60 9月  29 19:30 SPECS
drwxr-xr-x. 2 root root   57 9月  29 19:21 SRPMS
```
其中SOURCES目录中就有源码的一些tar包
比如cinder-11.1.1.tar.gz
```
总用量 20668
-rw-rw-r--. 1 root root 10041327 9月  29 19:10 cinder-11.1.1.tar.gz
-rw-rw-r--. 1 root root      387 6月  19 00:52 cinder-dist.conf
-rw-rw-r--. 1 root root      134 6月  19 00:52 cinder.logrotate
-rw-rw-r--. 1 root root      112 6月  19 00:52 cinder-sudoers
-rw-rw-r--. 1 root root 10975664 6月   4 22:33 nova-16.1.4.tar.gz
-rw-rw-r--. 1 root root      752 6月   4 23:10 nova-dist.conf
-rw-rw-r--. 1 root root      249 6月   4 23:10 nova-ifc-template
-rw-rw-r--. 1 root root       95 6月   4 23:10 nova.logrotate
-rw-rw-r--. 1 root root      190 6月   4 23:10 nova_migration_authorized_keys
-rw-rw-r--. 1 root root      265 6月   4 23:10 nova_migration_identity
-rw-rw-r--. 1 root root      679 6月   4 23:10 nova_migration-rootwrap_cold_migration
-rw-rw-r--. 1 root root      121 6月   4 23:10 nova_migration-rootwrap.conf
-rw-rw-r--. 1 root root      217 6月   4 23:10 nova_migration-sudoers
-rw-rw-r--. 1 root root     2058 6月   4 23:10 nova-migration-wrapper
-rw-rw-r--. 1 root root      695 6月   4 23:10 nova-placement-api.conf
-rw-rw-r--. 1 root root      109 6月   4 23:10 nova-ssh-config
-rw-rw-r--. 1 root root      158 6月   4 23:10 nova-sudoers
-rw-rw-r--. 1 root root      343 6月  19 00:52 openstack-cinder-api.service
-rw-rw-r--. 1 root root      335 6月  19 00:52 openstack-cinder-backup.service
-rw-rw-r--. 1 root root      344 6月  19 00:52 openstack-cinder-scheduler.service
-rw-rw-r--. 1 root root      389 6月  19 00:52 openstack-cinder-volume.service
-rw-rw-r--. 1 root root      230 6月   4 23:10 openstack-nova-api.service
-rw-rw-r--. 1 root root      234 6月   4 23:10 openstack-nova-cells.service
-rw-rw-r--. 1 root root      302 6月   4 23:10 openstack-nova-compute.service
-rw-rw-r--. 1 root root      242 6月   4 23:10 openstack-nova-conductor.service
-rw-rw-r--. 1 root root      251 6月   4 23:10 openstack-nova-consoleauth.service
-rw-rw-r--. 1 root root      244 6月   4 23:10 openstack-nova-console.service
-rw-rw-r--. 1 root root      248 6月   4 23:10 openstack-nova-metadata-api.service
-rw-rw-r--. 1 root root      299 6月   4 23:10 openstack-nova-network.service
-rw-rw-r--. 1 root root      304 6月   4 23:10 openstack-nova-novncproxy.service
-rw-rw-r--. 1 root root       73 6月   4 23:10 openstack-nova-novncproxy.sysconfig
-rw-rw-r--. 1 root root      248 6月   4 23:10 openstack-nova-os-compute-api.service
-rw-rw-r--. 1 root root      242 6月   4 23:10 openstack-nova-scheduler.service
-rw-rw-r--. 1 root root      216 6月   4 23:10 openstack-nova-serialproxy.service
-rw-rw-r--. 1 root root      225 6月   4 23:10 openstack-nova-spicehtml5proxy.service
-rw-rw-r--. 1 root root      216 6月   4 23:10 openstack-nova-xvpvncproxy.service
-rw-rw-r--. 1 root root        4 6月   4 23:10 policy.json
```
我们所需要的源码就在这个tar中，我们可以对这个tar镜像解压，然后修改代码，然后使用tar -zcvf生成新的tar包，然后把新的tar包拷贝到SOURCES目录下，对原有的tar包进行替换
- 制作rpm包
在SOURCES同目录下还有一个SPECS，这个下面有一个spec文件，这个文件就是用来生成rpm包的
```
# ll
总用量 48
-rw-rw-r--. 1 root root 14614 9月  29 16:46 openstack-cinder.spec
-rw-rw-r--. 1 root root 29641 6月   4 23:10 openstack-nova.spec
# rpmbuild -ba openstack-cinder.spec
```
命令之后会在RPMS目录下面生成相关的rpm包
```
# ll
总用量 11012
-rw-r--r--. 1 root root   60636 9月  29 19:21 openstack-cinder-11.1.1-1.el7_tstack.noarch.rpm
-rw-r--r--. 1 root root 3943300 9月  29 19:21 openstack-cinder-doc-11.1.1-1.el7_tstack.noarch.rpm
-rw-r--r--. 1 root root 4229080 9月  29 19:21 python-cinder-11.1.1-1.el7_tstack.noarch.rpm
-rw-r--r--. 1 root root 3038588 9月  29 19:21 python-cinder-tests-11.1.1-1.el7_tstack.noarch.rpm

```
注:原生的tar解压的时候最好不要在windows下面操作，因为有些文件的执行权限会没有，大爱linux
