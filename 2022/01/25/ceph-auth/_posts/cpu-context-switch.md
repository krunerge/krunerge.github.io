---
title: linux性能优化系列（2）cpu上下文切换
date: 2019-01-05 17:21:32
tags: linux 性能
---

# 测试cpu上下文切换工具
—————————————————————————————————————————————————————————————————————————————
使用sysbench这个工具，这个工具是线程基准测试工具，用来测试线程，这样可以测试调度和cpu上下文切换
```
sysbench --num-threads=10 --max-time=300 --max-requests=10000000 --test=threads run
```

# 如何查看系统cpu上下文切换情况
—————————————————————————————————————————————————————————————————————————————
- 1.vmstat工具可以查看系统的cpu上下文切换
```
[root@con01 ~(keystone_admin)]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 8  0   2928 28614520 7066708 11676172    0    0     3    34    0    0  1  5 94  0  0
 4  0   2928 28616408 7066708 11676196    0    0     0    12 8510 19720  2 14 85  0  0
 2  0   2928 28615164 7066708 11676208    0    0     0   116 7205 7983  2 14 85  0  0
```

- 2.pidstat加-w
```
pidstat -wt -u 5
```