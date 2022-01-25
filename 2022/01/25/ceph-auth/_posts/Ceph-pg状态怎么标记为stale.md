---
title: Ceph pg状态怎么标记为stale
date: 2021-02-23 17:52:59
tags: Ceph
---

# 问题
Ceph中的pg是合适标记为stale状态的

# 分析
## 1.做了一个实验，把osd的beacan超时这是为5s（原来是900s）
```
ceph daemon mon.a config set mon_osd_report_timeout 5
```
这个命令之后ceph 的osd都会标记为down状态，同时ceph 的pg（我的pg是单pool单pg，并且只在一个osd上面）状态按理来说应该是stale+active+clean,但是我ceph -s看状态一致都是active + clean,
![2](2.png)
其他原因如下，我们这种把超时设置小了，其实osd的进程还在，没有挂掉，osd还是会定时向mgr上报pg的状态，而osd里面记录的状态还是active+clean（这里有peering过），ceph -s看到也是从mgr获取，也就是其实是mgr在一个时刻设置了stale，但是后来被osd上报的pg状态给刷掉了，日志如下
![1](1.png)
那osd为什么会还是active+clean呢，其实刚开始设置超时的时候，osd被标记为 down，收到mon osdmap更新（mgr设置pg为stale也是收到mon的osdmap更新），经过peering，pg状态恢复到active+clean，后面osd状态没有变过，也就是osdmap没有变化，所以这个osd的pg状态会一直维护为active+clean

## 2.pg状态被标记为stale是mgr做的事情
mon在osdmap变化之后，会发送osdmap给mgr，mgr通过notify_osdmap进行处理
```
void ClusterState::notify_osdmap(const OSDMap &osd_map)
{
  ....
  PGMapUpdater::check_down_pgs(osd_map, pg_map, true,
			       need_check_down_pg_osds, &pending_inc);

  ...
}

```
在这里会调用check_down_pgs检查
```
void PGMapUpdater::check_down_pgs(
    const OSDMap &osdmap,
    const PGMap &pg_map,
    bool check_all,
    const set<int>& need_check_down_pg_osds,
    PGMap::Incremental *pending_inc)
{
  ....
	  _try_mark_pg_stale(osdmap, pgid, stat, pending_inc);
  ....
}
```
_try_mark_pg_stale这个函数有修改
```
static void _try_mark_pg_stale(
  const OSDMap& osdmap,
  pg_t pgid,
  const pg_stat_t& cur,
  PGMap::Incremental *pending_inc)
{
    ....
    newstat->state |= PG_STATE_STALE;
    newstat->last_unstale = ceph_clock_now();
  }
}
```
并在这里修改mgr内存的pg状态，然后向mon一直会报，因为我们kill调的osd已经不再像mgr会报pg状态了
