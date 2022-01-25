---
title: Ceph mon rank由来
date: 2021-03-11 22:25:43
tags: Ceph
---

# 目的
本篇主要分析一下ceph mon rank的由来，比如我们一套3个mon的集群到底怎么对应rank的
```
ceph mon stat
e1: 3 mons at {ceph-7=192.168.69.32:6789/0,ceph-8=192.168.69.57:6789/0,ceph-9=192.168.69.19:6789/0}, election epoch 242,ceph-8
```

# 分析
在Monmap中的calc_ranks()函数中有解析
```
void MonMap::calc_ranks() {

  ranks.resize(mon_info.size());
  addr_mons.clear();

  set<mon_info_t, rank_cmp> tmp;

  for (map<string,mon_info_t>::iterator p = mon_info.begin();
      p != mon_info.end();
      ++p) {
    mon_info_t &m = p->second;
    tmp.insert(m);

    // populate addr_mons
    assert(addr_mons.count(m.public_addr) == 0);
    addr_mons[m.public_addr] = m.name;
  }

  // map the set to the actual ranks etc
  unsigned i = 0;
  for (set<mon_info_t>::iterator p = tmp.begin();
      p != tmp.end();
      ++p, ++i) {
    ranks[i] = p->name;
  }
}
```
从上面可以看出，首先遍历mon_info这个map
```
map<string, mon_info_t> mon_info;
```
然后加入到临时的set<mon_info_t, rank_cmp> tmp;集合中从定义了比较方法
```
struct rank_cmp {
    bool operator()(const mon_info_t &a, const mon_info_t &b) const {
      if (a.public_addr == b.public_addr)
        return a.name < b.name;
      return a.public_addr < b.public_addr;
    }
  };
```
可以看出这个集合是首先按ip进行比较，然后在按名字进行比较，从小到大排序, 然后在然后ranks数组中
```
vector<string> ranks;
```
数组的下标就是ranks的值，所以可以看到ip从小到大分别是rank 0，1，2。
