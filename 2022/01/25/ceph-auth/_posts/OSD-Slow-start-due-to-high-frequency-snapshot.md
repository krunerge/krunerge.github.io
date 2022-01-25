---
title: 频繁的创建删除快照导致OSD启动很慢
date: 2020-03-30 10:00:29
tags: Ceph osd
---

# 一.问题
最近有一个环境出现OSD启动很慢，要40多分钟，很奇怪，看一下osd的日志发现一直在刷类似下面的日志
```
2020-04-05 19:52:59.925 7fa86b863700 20 PGPool::update cached_removed_snaps [1~f5,f7~1,f9~7,101~1,103~12d,233~ea,31e~1,320~64,385~1e,3a4~16b,510~7e,58f~4e,5de~49,629~8,632~28,65b~31,68e~a,699~c,6a6~6,6ae~38,6e8~27,711~2a,73c~1d,75a~34,790~4,795~17,7ad~22,7d1~10,7e2~41,824~28,84d~23,871~53,8c5~17,8de~18,8fa~42,93d~a,949~10,95a~a,965~32,999~1e,9b8~6,9bf~32,9f3~e,a02~40,a46~d,a54~9,a62~279,cdc~12b,e09~26,e30~2,e33~d,e42~e,e51~a,e5d~8,e67~4,e6c~1e,e8d~1b,ea9~1,eab~3,eaf~24,ed4~aa,f7f~35,fb6~4,fbc~c,fc9~c1,108b~5c,10e8~8,10f4~59,114f~4,1154~27,117c~1,117e~2,1181~2a,11ac~26,11d4~4,11d9~22,11fd~4,1202~58,125c~8,1265~2,1268~16,127f~40] newly_removed_snaps [] snapc 12be=[] (no change)
```
这个只是一个小环境上面的，真实环境上面上面的一条日志记录占了显示屏的两页，这是什么？我是谁，我在哪里，我要干什么？好吧，出现了关键字snap，感觉跟快照有点关系。下面分析一下

# 二.实验
分析这个问题，首先我们要知道上面的一坨类似1-f5的是什么，我们先来做个试验，在我的一个开发环境模拟一下创建快照和删除快照到底发生了什么，Let go
- 1.创建一个池test
```
ceph osd pool create test 16 16
```
- 2.创建两个卷test1和test2
```
[root@openstack-ceph01 ~]# rbd create test/test1 --size 1G
[root@openstack-ceph01 ~]# rbd create test/test2 --size 1G
```
- 3.创建快照前观察一下OSDMap版本
```
[root@openstack-ceph01 ~]# ceph osd dump
epoch 23303
...
```
osdmap的版本是23303,下面我们给test1创建一个快照snap4,至于为什么叫snap4，等话就知道

- 4.test1创建快照snap4
```
[root@openstack-ceph01 ~]# rbd snap create test/test1@snap4
```
查看一下快照信息
```
[root@openstack-ceph01 ~]# rbd snap ls test/test1
SNAPID NAME  SIZE  PROTECTED TIMESTAMP                
     4 snap4 1 GiB           Sun Apr  5 20:29:12 2020
```
快照id是4，不是从1或者0开始，这就是为什么我们创建快照的时候取名叫snap4，再看一下osdmap版本
```
[root@openstack-con01 ceph(keystone_admin)]# ceph osd dump
epoch 23304
...
```
版本变成了23304，也就是说创建快照osdmap版本会变化，为什么创建快照osdmap版本会变化呢，我等话解释，既然osdmap变化了，那我们看看osdmap到底变化了什么，对，你跟我想的一眼，查看一下osdmap的增量变化以及全量变化，我们知道osdmap变化之后会mon通知给osd，那我们去osd的数据目录下面meta目录看一下osdmap
```
[root@openstack-ceph01 meta]# find . -name *osdmap.23304*
./DIR_9/DIR_A/osdmap.23304__0_4EBB24A9__none
./DIR_E/DIR_0/inc\uosdmap.23304__0_46AAFD0E__none
```
由于分目录保存，所以使用上面查找，找到之后使用ceph-dencoder反序列化一下，因为这个里面存储的OSDMap或者OSDMap::Incremental类实例的序列化结果,让我们先看一下osdmap增量
- 4.1 osdmap增量
```
[root@openstack-ceph01 meta]# ceph-dencoder type OSDMap::Incremental import ./DIR_E/DIR_0/inc\\uosdmap.23304__0_46AAFD0E__none decode dump_json
{
    "epoch": 23304,
    ...
    "new_pools": [
        {
            ....
            "pool": 30,
            "snap_mode": "selfmanaged",
            "snap_seq": 4,
            "snap_epoch": 23304,
            "pool_snaps": [],
            "removed_snaps": "[1~3]",
            ...
        }
    ...
}
```
我们只关注快照相关的信息，可以看到，主要有上面的5个字段，快照seq是4，快照模式是selfmanaged，removed_snaps
- 4.2 快照模式
我们知道ceph的快照有两种模式，一种是对池对快照，一种就是这里的selfmanaged，也就是我们常用的对rbd卷的快照，ceph为了代码兼容处理这两种快照,把快照属性放在了pool池上面，快照作为池的一个特性，而pool的变化是要记录在osdmap里面的，因为客户端端在通过crush进行osd计算的时候需要通过找到池的相应信息，所以这里就解释了为什么创建快照osdmap版本会变化，因为pool变化了，上面是osdmap的增量数据，我们再看一下osdmap的全量数据，

- 4.3 osdmap全量
```
[root@openstack-ceph01 meta]# ceph-dencoder type OSDMap import ./DIR_9/DIR_A/osdmap.23304__0_4EBB24A9__none decode dump_json
{
    "epoch": 23304,
    ...
    "pools": [
        ...
        {
            "pool": 30,
            ...
            "snap_mode": "selfmanaged",
            "snap_seq": 4,
            "snap_epoch": 23304,
            "pool_snaps": [],
            "removed_snaps": "[1~3]",
            ...
        }
    ...
}
```
可以看到快照部分增量和全量是一样的

- 4.4 快照id在pool内部全局唯一
我们继续给test2创建快照snap5，在给test1创建快照snap6，看这名字聪明的你就知道我要做什么了，看一下操作结果
```
[root@openstack-ceph01 meta]# rbd snap create  test/test2@snap5
[root@openstack-ceph01 meta]# rbd snap ls test/test2
SNAPID NAME  SIZE  PROTECTED TIMESTAMP                
     5 snap5 1 GiB           Sun Apr  5 20:50:44 2020
[root@openstack-ceph01 meta]# rbd snap create  test/test1@snap6
[root@openstack-ceph01 meta]# rbd snap ls test/test1
SNAPID NAME  SIZE  PROTECTED TIMESTAMP                
     4 snap4 1 GiB           Sun Apr  5 20:29:12 2020
     6 snap6 1 GiB           Sun Apr  5 20:51:16 2020
```
可以从结果看到，看这id是从4开始，1、2、3快照id好像被预留了，而且一个池内所有卷的快照的id是单调递增的。这是我们再看一下osdmap
```
[root@openstack-ceph01 meta]# ceph osd dump
epoch 23306
...
```
osdmap已经是23306，版本增加了2，因为创建了两次快照，再看一下osdmap增量信息
```
[root@openstack-ceph01 meta]# ceph-dencoder type OSDMap::Incremental import ./DIR_E/DIR_6/inc\\uosdmap.23306__0_46AAF26E__none decode dump_json
{
    "epoch": 23306,
    ...
    "new_pools": [
        {
            "pool": 30,
            ...
            "snap_mode": "selfmanaged",
            "snap_seq": 6,
            "snap_epoch": 23306,
            "pool_snaps": [],
            "removed_snaps": "[1~3]",
            ...
       }
    ...
}
```
从上面看出好像就只修改了snap_seq，和snap_epoch，其他都没变，尤其是removed_snaps，这个到底是什么呢?

- 4.5 removed_snaps是个啥
我们前面已经知道一个池的快照id是从4开始的，1、2、3快照id是保留，或者已经删除（从这个字段的名字可以看出），那为什么要这个呢，小甲大胆猜测一下，是为了知道我现在还存在的快照的id，你看我们通过snap_seq知道最新的快照id为6，而我们通过removed_snaps知道已经删除了1-3快照id的快照，那就是说我们还有4、5、6id的快照，也就是通过排除已经删除的快照，得到还存在的快照，这个逻辑我想开发者是这么认为快照会经常打，但是删除比较少，所以通过记录删除的id反推存在的快照，这样记录的数据成本比较少，通过查看代码我发现removed_snap是一个间断集合
```
struct PGPool {

  ...
  interval_set<snapid_t> cached_removed_snaps;
  ...
}
```
学过数据集合的你看到这个会有什么反应，既然叫间断集合，那removed_snaps字段的1~3表示的应该是（start，len）也就是从id为1开始，后面连续三个id已经被删除，也就是1、2、3快照id被删除。这里还要提一下，这里的start和len是16进制显示的，比如108b~5c。

那是不是真的是这样呢，我们在给test1创建几个快照然后删除一个快照试试

- 5. test1删除快照
- 5.1 test1有创建了快照snap7，snap8，snap9三个快照
```
[root@openstack-ceph01 meta]# rbd snap create test/test1@snap7
[root@openstack-ceph01 meta]# rbd snap create test/test1@snap8
[root@openstack-ceph01 meta]# rbd snap create test/test1@snap9
[root@openstack-ceph01 meta]# rbd snap ls test/test1
SNAPID NAME  SIZE  PROTECTED TIMESTAMP                
     4 snap4 1 GiB           Sun Apr  5 20:29:12 2020
     6 snap6 1 GiB           Sun Apr  5 20:51:16 2020
     7 snap7 1 GiB           Sun Apr  5 21:14:22 2020
     8 snap8 1 GiB           Sun Apr  5 21:14:25 2020
     9 snap9 1 GiB           Sun Apr  5 21:14:36 2020
```
可以看到最新的快照id是9，看一下osdmap增量信息
```
[root@openstack-ceph01 meta]# ceph-dencoder type OSDMap::Incremental import ./DIR_E/DIR_9/inc\\uosdmap.23309__0_46AAF19E__none decode dump_json
{
    "epoch": 23309,
    ...
    "new_pools": [
        {
            "pool": 30,
            ...
            "snap_mode": "selfmanaged",
            "snap_seq": 9,
            "snap_epoch": 23309,
            "pool_snaps": [],
            "removed_snaps": "[1~3]",
            ...
        }
    ...
}            
```
看到removed_snaps没有变化

- 5.2 删除test1的快照snap8
```
[root@openstack-ceph01 meta]# rbd snap rm test/test1@snap8
Removing snap: 100% complete...done.
[root@openstack-ceph01 meta]# rbd snap ls test/test1
SNAPID NAME  SIZE  PROTECTED TIMESTAMP                
     4 snap4 1 GiB           Sun Apr  5 20:29:12 2020
     6 snap6 1 GiB           Sun Apr  5 20:51:16 2020
     7 snap7 1 GiB           Sun Apr  5 21:14:22 2020
     9 snap9 1 GiB           Sun Apr  5 21:14:36 2020
```
test1的snap8快照已经删除，也就是id为8的快照删除了，在我们看osdmap增量之前，我们大胆的猜测一下removed_snaps会变成什么，我赌10块钱会变成[1~3, 8~1], 好，我们看一下
```
[root@openstack-ceph01 meta]# ceph-dencoder type OSDMap::Incremental import ./DIR_E/DIR_F/inc\\uosdmap.23310__0_46AAF6FE__none decode dump_json
{
    "epoch": 23310,
    ...
    "new_pools": [
        {
            "pool": 30,
            ...
            "snap_mode": "selfmanaged",
            "snap_seq": 10,
            "snap_epoch": 23310,
            "pool_snaps": [],
            "removed_snaps": "[1~3,8~1,a~1]",
            ...
       }
    ...
}
```
removed_snaps既然变成了[1~3,8~1,a~1]，其中8~1是情理之中，而a~1这个意料之外是个啥，我们知道这个是16进制，变成10进制就是10~1，也就是删除了快照id 10，快照10哪里来的，我们只创建到了快照9，咦，snap_seq竟然变成了10，嗯。。。待我冷静一下，删除快照，快照的seq变成10，增加了1，为什么？我们在品一品这个字段的名字snap_seq，seq序号，他不是id，小甲大胆猜一下，这个应该是快照的版本，创建快照和删除快照都会修改版本，而快照id只是通过版本值演变而来，这样一来就说的通了，也就是删除快照，快照版本增加，但是在removed_snaps中会把这个版本去掉，因为确实是没有这个id的快照的，既然这么说，你猜猜下面再创建一个快照，快照的id会是多少，我觉得会是11，试一下
```
[root@openstack-ceph01 meta]# rbd snap create test/test1@snap11
[root@openstack-ceph01 meta]# rbd snap ls test/test1
SNAPID NAME   SIZE  PROTECTED TIMESTAMP                
     4 snap4  1 GiB           Sun Apr  5 20:29:12 2020
     6 snap6  1 GiB           Sun Apr  5 20:51:16 2020
     7 snap7  1 GiB           Sun Apr  5 21:14:22 2020
     9 snap9  1 GiB           Sun Apr  5 21:14:36 2020
    11 snap11 1 GiB           Sun Apr  5 21:30:38 2020
```
果然如此。

通过上面的实验分析我们知道了快照的一些基本知识，也知道osd启动慢时，一直刷的那一坨是个啥，那为什么为一直刷导致osd启动慢呢？

# 三.分析
osd启动的时候会加载osdmap，pool、osd、快照的变化都记录在了osdmap里面，通过加载osdmap来在osd内存中构造相关的数据，大致是这个逻辑，通过代码找到打印这个日志的地方
```
2020-04-05 19:52:59.925 7fa86b863700 20 PGPool::update cached_removed_snaps [1~f5,f7~1,f9~7,101~1,103~12d,233~ea,31e~1,320~64,385~1e,3a4~16b,510~7e,58f~4e,5de~49,629~8,632~28,65b~31,68e~a,699~c,6a6~6,6ae~38,6e8~27,711~2a,73c~1d,75a~34,790~4,795~17,7ad~22,7d1~10,7e2~41,824~28,84d~23,871~53,8c5~17,8de~18,8fa~42,93d~a,949~10,95a~a,965~32,999~1e,9b8~6,9bf~32,9f3~e,a02~40,a46~d,a54~9,a62~279,cdc~12b,e09~26,e30~2,e33~d,e42~e,e51~a,e5d~8,e67~4,e6c~1e,e8d~1b,ea9~1,eab~3,eaf~24,ed4~aa,f7f~35,fb6~4,fbc~c,fc9~c1,108b~5c,10e8~8,10f4~59,114f~4,1154~27,117c~1,117e~2,1181~2a,11ac~26,11d4~4,11d9~22,11fd~4,1202~58,125c~8,1265~2,1268~16,127f~40] newly_removed_snaps [] snapc 12be=[] (no change)
```
代码是这个函数打印的
```
void PGPool::update(OSDMapRef map)
{
  ...
  if ((map->get_epoch() != cached_epoch + 1) ||
      (pi->get_snap_epoch() == map->get_epoch())) {
    updated = true;
    pi->build_removed_snaps(newly_removed_snaps);
    interval_set<snapid_t> intersection;
    intersection.intersection_of(newly_removed_snaps, cached_removed_snaps);
    if (intersection == cached_removed_snaps) {
        newly_removed_snaps.subtract(cached_removed_snaps);
        cached_removed_snaps.union_of(newly_removed_snaps);
    } else {
        lgeneric_subdout(g_ceph_context, osd, 0) << __func__
          << " cached_removed_snaps shrank from " << cached_removed_snaps
          << " to " << newly_removed_snaps << dendl;
        cached_removed_snaps = newly_removed_snaps;
        newly_removed_snaps.clear();
    }
    snapc = pi->get_snap_context();
  } else {
    ...
    newly_removed_snaps.clear();
  }
  cached_epoch = map->get_epoch();
  lgeneric_subdout(g_ceph_context, osd, 20)
    << "PGPool::update cached_removed_snaps "
    << cached_removed_snaps
    << " newly_removed_snaps "
    << newly_removed_snaps
    << " snapc " << snapc
    << (updated ? " (updated)":" (no change)")
    << dendl;
}
```
上面是ceph 10.2.7版本的代码，这个函数是通过osdmap更新pool信息，在最后打印输出的上面，可以看到对removed_snaps这个间断结合做合并等操作，也就是对集合进行类似交集的操作，我之前也说启动慢的环境，removed_snaps输出的集合两页屏幕都没打印完，这是由于频繁打快照删除快照导致removed_snaps集合很大，这样集合做操作耗时会比较慢，这还只是单个osdmap操作。
我们之前启动慢的环境由于其他原因导致osd集群不断的在up和down，而我们知道这都是要修改osdmap的，这样导致我们的osdmap版本很大，这样就导致我们osd启动慢，两个原因
```
原因1：osdmap版本很大，要加载的osdmap很多
原因2：removed_snaps集合很大，加载一次osdmap时间很长
```

# 四.有没有什么改进方法
其实通过上面的实验发现，我们看到在创建快照的时候，其实osdmap只是修改了snap_seq,removed_snaps没有变化，只有在删除的时候removed_snaps才会改变，是不是可以去掉removed_snaps没变时osdmap的加载，嗯，好像很有道理，其实社区已经做了修改,看一下代码
```
void PGPool::update(OSDMapRef map)
{
  ...
  if ((map->get_epoch() != cached_epoch + 1) ||
      (pi->get_snap_epoch() == map->get_epoch())) {
    updated = true;
    if (pi->maybe_updated_removed_snaps(cached_removed_snaps)) {  // 加了判断removed_snaps集合是否变化，没变化不做集合操作
      pi->build_removed_snaps(newly_removed_snaps);
      interval_set<snapid_t> intersection;
      intersection.intersection_of(newly_removed_snaps, cached_removed_snaps);
      if (intersection == cached_removed_snaps) {
          cached_removed_snaps.swap(newly_removed_snaps);
          newly_removed_snaps = cached_removed_snaps;
          newly_removed_snaps.subtract(intersection);
      } else {
          lgeneric_subdout(cct, osd, 0) << __func__
            << " cached_removed_snaps shrank from " << cached_removed_snaps
            << " to " << newly_removed_snaps << dendl;
          cached_removed_snaps.swap(newly_removed_snaps);
          newly_removed_snaps.clear();
      }
    } else
      newly_removed_snaps.clear();
    snapc = pi->get_snap_context();
  } else {
    ....
    newly_removed_snaps.clear();
  }
  cached_epoch = map->get_epoch();
  lgeneric_subdout(cct, osd, 20)
    << "PGPool::update cached_removed_snaps "
    << cached_removed_snaps
    << " newly_removed_snaps "
    << newly_removed_snaps
    << " snapc " << snapc
    << (updated ? " (updated)":" (no change)")
    << dendl;
}
```
maybe_updated_removed_snaps这个判断removed_snaps集合是否变化
```
bool pg_pool_t::maybe_updated_removed_snaps(const interval_set<snapid_t>& cached) const
{
  if (is_unmanaged_snaps_mode()) { // remove_unmanaged_snap increments range_end
    if (removed_snaps.empty() || cached.empty()) // range_end is undefined
      return removed_snaps.empty() != cached.empty();
    return removed_snaps.range_end() != cached.range_end();
  }
  return true;
}
```
上面是ceph 12.2.10版本加了判断removed_snaps集合是否变化，没变化不做集合操作
