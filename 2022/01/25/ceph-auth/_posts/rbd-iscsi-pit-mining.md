---
title: rbd iscsi 采坑
date: 2020-02-26 16:16:10
tags: rbd iscsi python
---

# 坑
- 1.target的创建时create命令，删除是clearconfig
- 2.tpg有创建，没有删除接口
- 3.创建新gateway没有进行同步
- 4.lun在后端ceph的创建者，只有lun的owner节点
