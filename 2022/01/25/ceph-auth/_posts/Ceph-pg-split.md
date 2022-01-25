---
title: Ceph pg split发生在什么时候
date: 2021-02-23 20:48:27
tags: Ceph
---

# 问题
之前一直好奇Ceph pg split发生在什么时候

# 分析
pg的分裂是在peering过程进行的

命令修改osdmap ，mon发散osdmap，然后通过peering advmap同步pg 的osdmap，在peering的时候会给每个pg进行osdmap的更新，更新的时候会判断是不是要split分裂，代码如下
```
bool OSD::advance_pg(
  epoch_t osd_epoch, PG *pg,
  ThreadPool::TPHandle &handle,
  PG::RecoveryCtx *rctx,
  set<PGRef> *new_pgs)
{
  ....
  // Check for split!
    set<spg_t> children;
    spg_t parent(pg->info.pgid);
    if (parent.is_split(
	lastmap->get_pg_num(pg->pool.id),
	nextmap->get_pg_num(pg->pool.id),
	&children)) {
      service.mark_split_in_progress(pg->info.pgid, children);
      split_pgs(
	pg, children, new_pgs, lastmap, nextmap,
	rctx);
    }
  ....
}
```
