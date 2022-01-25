---
title: rbd的watcher数据持久化在哪里
date: 2020-03-15 13:50:32
tags: rbd
---

# 问题
最近在看一个问题，需要查看rbd watcher相关信息，于是便想看一下watcher存在了哪里

# 分析
- 通过rados listwatchers命令我们知道watcher是存在osd服务那边的object_info_t结构体中，那要查watcher存在哪里，就是要查object_info_t存在哪里

- 在filestore的存储引擎下，object_info_t存在rbd_header文件对象的xattr扩展属性中

- 步骤如下：
- 1. 获取rbd的header对象
```
[root@con01 tmp(keystone_admin)]# rbd info rbd/gkk1
rbd image 'gkk1':
	size 1024 MB in 256 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.301027238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags:
```
header就是rbd_header.301027238e1f29

- 2. 查找header对象存在哪些osd上面
```
[root@con01 3.ee_head(keystone_admin)]# ceph osd map rbd rbd_header.301027238e1f29
osdmap e22962 pool 'rbd' (0) object 'rbd_header.301027238e1f29' -> pg 0.a5bd3cee (0.ee) -> up ([3,6,10], p3) acting ([3,6,10], p3
```
三个osd都会有，我们找主OSD--3

- 3. 在filestore下找到rbd_header对象
```
[root@con02 0.ee_head]# ll
total 4096
-rw-r--r-- 1 ceph ceph       0 Jan 16  2019 __head_000000EE__0
-rw-r--r-- 1 ceph ceph 4194304 Mar 14 15:01 rbd\udata.301027238e1f29.0000000000000063__head_B1E6FFEE__0
-rw-r--r-- 1 ceph ceph       0 Jan 17  2019 rbd\uheader.301027238e1f29__head_A5BD3CEE__0
```

- 4. 列举扩展属性
```
[root@con02 0.ee_head]# getfattr rbd\\uheader.301027238e1f29__head_A5BD3CEE__0
# file: rbd\134uheader.301027238e1f29__head_A5BD3CEE__0
user.ceph._
user.ceph._@1
user.ceph._lock.rbd_lock
user.ceph.snapset
user.cephos.spill_out
```

- 5. object_info_t就存在以user.ceph.\_开头的扩展属性里面，如下获取
```
[root@con02 0.ee_head]# attr -q -g "ceph._" rbd\\uheader.301027238e1f29__head_A5BD3CEE__0   > /root/object_info_t.txt
[root@con02 0.ee_head]# attr -q -g "ceph._@1" rbd\\uheader.301027238e1f29__head_A5BD3CEE__0   >> /root/object_info_t.txt
```
其中@1可以看出object_info_t会由于大小分成好几个扩展属性进行存取

- 6. 通过ceph-dencode进行解析
```
[root@con02 0.ee_head]# ceph-dencoder type object_info_t import /root/object_info_t.txt decode dump_json
{
    "oid": {
        "oid": "rbd_header.301027238e1f29",
        "key": "",
        "snapid": -2,
        "hash": 2780642542,
        "max": 0,
        "pool": 0,
        "namespace": ""
    },
    "version": "22962'4735",
    "prior_version": "22962'4734",
    "last_reqid": "client.8079633.0:10",
    "user_version": 4735,
    "size": 0,
    "mtime": "2020-03-15 14:38:36.329259",
    "local_mtime": "2020-03-15 14:38:36.321500",
    "lost": 0,
    "flags": 60,
    "snaps": [],
    "truncate_seq": 0,
    "truncate_size": 0,
    "data_digest": 4294967295,
    "omap_digest": 3148395953,
    "watchers": {
        "client.8079633": {
            "cookie": 139996606507840,
            "timeout_seconds": 30,
            "addr": {
                "nonce": 3613298630,
                "addr": "10.125.232.77:0"
            }
        }
    }
}
```
从上面我们就看到了watcher的内容

# 这里有一个区别要提一下
我们通过对rbd_header文件对象获取扩展属性可以看到
```
[root@con02 0.ee_head]# getfattr rbd\\uheader.301027238e1f29__head_A5BD3CEE__0
# file: rbd\134uheader.301027238e1f29__head_A5BD3CEE__0
user.ceph._
user.ceph._@1
user.ceph._lock.rbd_lock
user.ceph.snapset
user.cephos.spill_out
```

我们通过rados listxattr可以看到如属性
```[root@con02 0.ee_head]# rados listxattr -p rbd rbd_header.301027238e1f29
lock.rbd_lock

```
可以看到只有一个rbd_lock是相同的，刚开始我也以为是在这个里面能找到watcher，其实rbd_lock是属于rbd的属性，而其他属于rados层公共属性
