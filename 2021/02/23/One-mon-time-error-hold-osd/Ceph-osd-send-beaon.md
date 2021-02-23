---
title: Ceph osd发送beaon请求分析
date: 2021-02-23 20:22:33
tags: Ceph
---

# 问题
在调试mon的时候，发现osd被标记为down了，为什么？

# 分析
- 1.ceph的osd每隔5分钟会向mon上报一次
```
void OSD::tick_without_osd_lock()
{
  ....
  if (is_active()) {
    if (!scrub_random_backoff()) {
      sched_scrub();
    }
    service.promote_throttle_recalibrate();
    resume_creating_pg();
    bool need_send_beacon = false;
    const auto now = ceph::coarse_mono_clock::now();
    {
      // borrow lec lock to pretect last_sent_beacon from changing
      Mutex::Locker l{min_last_epoch_clean_lock};
      const auto elapsed = now - last_sent_beacon;
      if (chrono::duration_cast<chrono::seconds>(elapsed).count() >
        cct->_conf->osd_beacon_report_interval) {
        need_send_beacon = true;
      }
    }
    if (need_send_beacon) {
      send_beacon(now);
    }
  }
  ....
}
```
小甲在给mon打断点的时候，其实是stop了进程，那mon那边是不会处理这个请求的，也就会导致mon里面记录的osd report时间不会更新，最后导致osd超过15分钟被标记为down

- 2.mon的osdmonitor会周期性的tick会调用handle_osd_timeouts
```
void OSDMonitor::tick()
{
  ...
  if (osdmap.require_osd_release >= CEPH_RELEASE_LUMINOUS &&
      mon->monmap->get_required_features().contains_all(
	ceph::features::mon::FEATURE_LUMINOUS)) {
    if (handle_osd_timeouts(now, last_osd_report)) {
      do_propose = true;
    }
  }
  ...
}
```
检查beacon，超时标记down
```
bool OSDMonitor::handle_osd_timeouts(const utime_t &now,
				     std::map<int,utime_t> &last_osd_report)
{
  ...
  for (int i=0; i < max_osd; ++i) {
  ...
    } else if (can_mark_down(i)) {
      utime_t diff = now - t->second;
      if (diff > timeo) {
	mon->clog->info() << "osd." << i << " marked down after no beacon for "
			  << diff << " seconds";
	derr << "no beacon from osd." << i << " since " << t->second
	     << ", " << diff << " seconds ago.  marking down" << dendl;
	pending_inc.new_state[i] = CEPH_OSD_UP;
	new_down = true;
      }
    }
  }
  return new_down;
}
```

- 3.其实osd的进程还是在的，还是会发送beacon，但是在osd被mon标记down之后，后面osd发送的beacon，mon已经不再处理了
```
bool OSDMonitor::prepare_beacon(MonOpRequestRef op)
{
  ....
  if (!src.is_osd() ||
      !osdmap.is_up(from) ||
      beacon->get_orig_source_inst() != osdmap.get_inst(from)) {
    dout(1) << " ignoring beacon from non-active osd." << dendl;
    return false;
  }

 ....
}
```
所以在我取消mon的断点之后，即使osd还是发送beacon，mon也不会在标记osd为up
