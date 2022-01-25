---
title: Ceph中up set和acting set的区别
date: 2021-02-23 20:53:55
tags: Ceph
---

# 问题
Ceph中up set和acting set的区别，它们到底是怎么获得的

# 分析
在Crush计算那块得到
```
void OSDMap::_pg_to_up_acting_osds(
  const pg_t& pg, vector<int> *up, int *up_primary,
  vector<int> *acting, int *acting_primary,
  bool raw_pg_to_pg) const
{
  ....
  _get_temp_osds(*pool, pg, &_acting, &_acting_primary);
 if (_acting.empty() || up || up_primary) {
   _pg_to_raw_osds(*pool, pg, &raw, &pps);
   _apply_upmap(*pool, pg, &raw);
   _raw_to_up_osds(*pool, raw, &_up);
   _up_primary = _pick_primary(_up);
   _apply_primary_affinity(pps, *pool, &_up, &_up_primary);
   if (_acting.empty()) {
     _acting = _up;
     if (_acting_primary == -1) {
       _acting_primary = _up_primary;
     }
   }

   if (up)
     up->swap(_up);
   if (up_primary)
     *up_primary = _up_primary;
 }
  ....
}
```
上面这段可以大致看出crush计算出来的步骤
- 首先算出的是raw集合
- 然后通过upmap调整
- 然后在去掉不是up的
- 先算出up集合，然后在算出acting集合
可以看出：如果有pg temp，acting就是pg temp，否者acting等于up
