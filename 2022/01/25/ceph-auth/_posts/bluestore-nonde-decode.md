---
title: 解析bluestore onode
date: 2021-02-22 22:42:38
tags: ceph
---

# 问题
解析bluestore模式下面的Onode

# 方法
## 1. 首先通过ceph-kv-tool获取对应的元数据
```
./bin/ceph-kvstore-tool   bluestore-kv dev/osd0 get O %7f%80%00%00%00%00%00%00%02%eaE%e1F%21tangmi%21%3d%ff%ff%ff%ff%ff%ff%ff%fe%ff%ff%ff%ff%ff%ff%ff%ffo  out /home/krunerge/onode
```

## 2. 这个获取的数据不止包括Onode，我们可以通过写时候的osd日志可以看出
```
2021-02-08 14:55:21.519947 7f5b11c7d700 20 bluestore(/data/ceph/ceph-aduit-log/Ceph/build/dev/osd0)   onode #2:ea45e146:::tangmi:head# is 506 (344 bytes onode + 2 bytes spanning blobs + 160 bytes inline extents)
```
这个里面有344个字节是Onode结构

## 3. dd获取
```
dd if=/home/krunerge/onode of=/home/krunerge/onode_tmp bs=1 count=344
```

## 4. ceph-dencoder解析
```
./bin/ceph-dencoder type bluestore_onode_t import  /home/krunerge/onode_tmp decode dump_json
{
    "nid": 53306,
    "size": 131072,
    "attrs": {
        "attr": {
            "name": "_",
            "len": 261
        },
        "attr": {
            "name": "snapset",
            "len": 35
        }
    },
    "flags": "",
    "extent_map_shards": [],
    "expected_object_size": 0,
    "expected_write_size": 0,
    "alloc_hint_flags": 0
}
```
