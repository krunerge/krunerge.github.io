---
title: tcmu-runner编译
date: 2020-04-05 19:01:02
tags: C tcmu-runner
---

# tcmu-runner编译
之前使用的是tcmu-runner的1.3.0版本，看了一下github上面最新的是1.5.2准备拿来更新一下，看看有没有什么性能的优化

# 源码编译
- 1.下载源码包
这个可以直接在github上面下载
- 2.编译有问题
```
[root@master ~/tcmu-runner/tcmu-runner-1.5.2]# make
[  2%] Building C object CMakeFiles/tcmu.dir/libtcmu.c.o
/root/tcmu-runner/tcmu-runner-1.5.2/libtcmu.c:38:37: error: 'NLA_S32' undeclared here (not in a function)
  [TCMU_ATTR_CMD_STATUS] = { .type = NLA_S32 },
                                     ^
/root/tcmu-runner/tcmu-runner-1.5.2/libtcmu.c: In function 'send_netlink_reply':
/root/tcmu-runner/tcmu-runner-1.5.2/libtcmu.c:100:2: error: implicit declaration of function 'nla_put_s32' [-Werror=implicit-function-declaration]
  ret = nla_put_s32(msg, TCMU_ATTR_CMD_STATUS, status);
  ^
cc1: all warnings being treated as errors
make[2]: *** [CMakeFiles/tcmu.dir/libtcmu.c.o] Error 1
make[1]: *** [CMakeFiles/tcmu.dir/all] Error 2
make: *** [all] Error 2
```
说没有NLA_S32和nla_put_s32定义，看了一下代码，有头文件导入
```
#include <libnl3/netlink/genl/genl.h>
#include <libnl3/netlink/genl/mngt.h>
#include <libnl3/netlink/genl/ctrl.h>
```
在系统头文件/usr/include进行搜索，发现应该在这个文件中
```
/usr/include/libnl3/netlink/attr.h
```
但是没有，该开始以为是内核的头文件，后来看一下是
```
[root@master ~/tcmu-runner/tcmu-runner-1.5.2]# rpm -qf  /usr/include/libnl3/netlink/attr.h
libnl3-devel-3.2.28-4.el7.x86_64
```
是这个包libnl3-devel版本有点低，代码中没有定义NLA_S32，这个是一个无符号32位，所以升级，上面的3.2.28便是我升级后的版本
- 3.编译
```
[root@master ~/tcmu-runner/tcmu-runner-1.5.2]# make
[  2%] Generating tcmuhandler-generated.c, tcmuhandler-generated.h
Scanning dependencies of target tcmu
[  5%] Building C object CMakeFiles/tcmu.dir/strlcpy.c.o
[  7%] Building C object CMakeFiles/tcmu.dir/configfs.c.o
[ 10%] Building C object CMakeFiles/tcmu.dir/api.c.o
[ 12%] Building C object CMakeFiles/tcmu.dir/libtcmu.c.o
[ 15%] Building C object CMakeFiles/tcmu.dir/libtcmu-register.c.o
[ 17%] Building C object CMakeFiles/tcmu.dir/tcmuhandler-generated.c.o
[ 20%] Building C object CMakeFiles/tcmu.dir/libtcmu_log.c.o
[ 23%] Building C object CMakeFiles/tcmu.dir/libtcmu_config.c.o
[ 25%] Building C object CMakeFiles/tcmu.dir/libtcmu_time.c.o
Linking C shared library libtcmu.so
[ 25%] Built target tcmu
[ 28%] Building C object CMakeFiles/consumer.dir/scsi.c.o
[ 30%] Building C object CMakeFiles/consumer.dir/consumer.c.o
Linking C executable consumer
[ 30%] Built target consumer
[ 33%] Building C object CMakeFiles/handler_file.dir/file_example.c.o
Linking C shared library handler_file.so
[ 33%] Built target handler_file
[ 35%] Building C object CMakeFiles/handler_file_optical.dir/scsi.c.o
[ 38%] Building C object CMakeFiles/handler_file_optical.dir/file_optical.c.o
Linking C shared library handler_file_optical.so
[ 38%] Built target handler_file_optical
[ 41%] Building C object CMakeFiles/handler_file_zbc.dir/scsi.c.o
[ 43%] Building C object CMakeFiles/handler_file_zbc.dir/file_zbc.c.o
Linking C shared library handler_file_zbc.so
[ 43%] Built target handler_file_zbc
[ 46%] Building C object CMakeFiles/handler_rbd.dir/rbd.c.o
Linking C shared library handler_rbd.so
[ 46%] Built target handler_rbd
Scanning dependencies of target tcmu-runner
[ 48%] Building C object CMakeFiles/tcmu-runner.dir/tcmur_cmd_handler.c.o
[ 51%] Building C object CMakeFiles/tcmu-runner.dir/tcmur_aio.c.o
[ 53%] Building C object CMakeFiles/tcmu-runner.dir/tcmur_device.c.o
[ 56%] Building C object CMakeFiles/tcmu-runner.dir/target.c.o
[ 58%] Building C object CMakeFiles/tcmu-runner.dir/alua.c.o
[ 61%] Building C object CMakeFiles/tcmu-runner.dir/scsi.c.o
[ 64%] Building C object CMakeFiles/tcmu-runner.dir/main.c.o
[ 66%] Building C object CMakeFiles/tcmu-runner.dir/tcmuhandler-generated.c.o
Linking C executable tcmu-runner
[ 69%] Built target tcmu-runner
[ 71%] Building C object CMakeFiles/tcmu-synthesizer.dir/scsi.c.o
[ 74%] Building C object CMakeFiles/tcmu-synthesizer.dir/tcmu-synthesizer.c.o
Linking C executable tcmu-synthesizer
[ 74%] Built target tcmu-synthesizer
Scanning dependencies of target tcmu_static
[ 76%] Building C object CMakeFiles/tcmu_static.dir/strlcpy.c.o
[ 79%] Building C object CMakeFiles/tcmu_static.dir/configfs.c.o
[ 82%] Building C object CMakeFiles/tcmu_static.dir/api.c.o
[ 84%] Building C object CMakeFiles/tcmu_static.dir/libtcmu.c.o
[ 87%] Building C object CMakeFiles/tcmu_static.dir/libtcmu-register.c.o
[ 89%] Building C object CMakeFiles/tcmu_static.dir/tcmuhandler-generated.c.o
[ 92%] Building C object CMakeFiles/tcmu_static.dir/libtcmu_log.c.o
[ 94%] Building C object CMakeFiles/tcmu_static.dir/libtcmu_config.c.o
[ 97%] Building C object CMakeFiles/tcmu_static.dir/libtcmu_time.c.o
Linking C static library libtcmu_static.a
[100%] Built target tcmu_static
```
make一下编译完成

# 更新二进制
ldd和pldd看一下tcmu-runner二进制和进程依赖哪些动态库，发现有libtcmu.so和handler_rbd.so，所以替换这两个动态库加上tcmu-runner二进制就可以
