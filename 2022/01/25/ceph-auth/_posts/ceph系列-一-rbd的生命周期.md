---
title: 'ceph系列(一): rbd的生命周期'
date: 2020-02-15 23:39:45
tags: ceph rbd
---

# rbd 创建
------------------------------------------------------------------
- 1. rbd的order的取值范围是多少
```
order [12,25]
```

- 2. rbd创建流程
```
1. 在没有rbd_info的时候创建rbd_info这个对象，这个对象的内容是"overwrite validated"
2. 创建rbd_id.{image_name},里面记录的内容是image id
3. 在没有rbd_diretory的时候创建rbd_directory，这个对象的omap中存了name到id的和id到name的映射
4. 对比客户端和服务端的feature
```

- 3. 一个pg最多可以映射到16个osd

- 4. rbd快照个数uint64

- 5. 在L版之后rbd的快照个数是可以限制的，有limit接口
