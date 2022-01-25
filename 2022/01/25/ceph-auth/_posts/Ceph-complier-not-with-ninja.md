---
title: Ceph complier not with ninja
date: 2021-11-25 10:28:25
tags: ceph 编译
---

# 回归make编译
现在高版本的ceph编译使用的是ninja，那如果你想回归make编译如何操作呢？

# 修改do_cmake.sh
```
PYBUILD="3"
-ARGS="-GNinja"
+ARGS=" "
```
这里把-GNinja改成空就可以
