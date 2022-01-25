---
title: k8s learn(6) 隔离与限制
date: 2018-10-03 15:50:35
tags: k8s 张磊
---

# 容器与虚拟机
![aa](1.png)
## 优势
- 性能：
vm: Guest Os的操作要经过Hypervisor的转换，再到宿主机，所以计算，存储，网络性能损耗比较严重
容器：因为本身就是宿主机的进程，损耗可以忽略不计
## 缺点
- namespace隔离没有虚拟机化技术彻底
1. 内核是宿主机的内核：window不能运行linux，低内核不能运行高内核
2. linux内核中有些资源是不能被namespace，比如时间
3. 共享内核会导致攻击面变大，虽然可以使用Seccomp进行系统的调用的加固，但是有两个问题：一是性能损耗，二是不知道应该给哪些系统调用加固

# Cgroups
主要用来限制CPU，内存，磁盘，网络带宽
Cgroups表现出来的接口是文件系统
- Cgroups位置
```
# ll /sys/fs/cgroup/
总用量 0
dr-xr-xr-x. 6 root root  0 9月  29 12:55 blkio
lrwxrwxrwx. 1 root root 11 9月  29 12:55 cpu -> cpu,cpuacct
lrwxrwxrwx. 1 root root 11 9月  29 12:55 cpuacct -> cpu,cpuacct
dr-xr-xr-x. 7 root root  0 9月  29 12:55 cpu,cpuacct
dr-xr-xr-x. 5 root root  0 9月  29 12:55 cpuset
dr-xr-xr-x. 6 root root  0 9月  29 12:55 devices
dr-xr-xr-x. 5 root root  0 9月  29 12:55 freezer
dr-xr-xr-x. 5 root root  0 9月  29 12:55 hugetlb
dr-xr-xr-x. 6 root root  0 9月  29 12:55 memory
lrwxrwxrwx. 1 root root 16 9月  29 12:55 net_cls -> net_cls,net_prio
dr-xr-xr-x. 5 root root  0 9月  29 12:55 net_cls,net_prio
lrwxrwxrwx. 1 root root 16 9月  29 12:55 net_prio -> net_cls,net_prio
dr-xr-xr-x. 5 root root  0 9月  29 12:55 perf_event
dr-xr-xr-x. 6 root root  0 9月  29 12:55 pids
dr-xr-xr-x. 2 root root  0 9月  29 12:55 rdma
dr-xr-xr-x. 6 root root  0 9月  29 12:55 systemd
```
## 使用示例
- 创建一个控制组,会自动生成资源控制文件
```
# cd /sys/fs/cgroup/cpu
# mkdir container
# ll container/
总用量 0
-rw-r--r--. 1 root root 0 10月  3 16:12 cgroup.clone_children
-rw-r--r--. 1 root root 0 10月  3 16:12 cgroup.procs
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.stat
-rw-r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage_all
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage_percpu
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage_percpu_sys
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage_percpu_user
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage_sys
-r--r--r--. 1 root root 0 10月  3 16:12 cpuacct.usage_user
-rw-r--r--. 1 root root 0 10月  3 16:12 cpu.cfs_period_us
-rw-r--r--. 1 root root 0 10月  3 16:12 cpu.cfs_quota_us
-rw-r--r--. 1 root root 0 10月  3 16:12 cpu.rt_period_us
-rw-r--r--. 1 root root 0 10月  3 16:12 cpu.rt_runtime_us
-rw-r--r--. 1 root root 0 10月  3 16:12 cpu.shares
-r--r--r--. 1 root root 0 10月  3 16:12 cpu.stat
-rw-r--r--. 1 root root 0 10月  3 16:12 notify_on_release
-rw-r--r--. 1 root root 0 10月  3 16:12 tasks
```
- 启动死循环
```
# while : ; do : ; done &
[1] 68322

# top
top - 16:17:50 up 4 days,  3:22, 12 users,  load average: 2.07, 1.83, 1.78
Tasks: 247 total,   2 running, 166 sleeping,   0 stopped,   0 zombie
%Cpu(s): 52.6 us,  0.8 sy,  0.0 ni, 46.2 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :  4016192 total,   838948 free,  1630224 used,  1547020 buff/cache
KiB Swap:  3014652 total,  2518980 free,   495672 used.  1774528 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                
 68322 root      20   0  117352   3092    768 R 100.0  0.1   0:14.79 bash                                                                                                                   
119028 root      20   0  297036 245732  55584 S  50.8  6.1  93:13.82 kube-controller                                                                                                        
118812 root      20   0  657528 513892  66756 S  43.5 12.8  92:29.19 kube-apiserver                                                                                                         
119031 root      20   0  103016  92976  29568 S  16.3  2.3  37:35.77 kube-scheduler                                                                                                         
119156 root      20   0  740200 102960  59764 S   2.0  2.6  10:55.70 kubelet                                                                                                                
  2837 root      20   0 1513220  58348  17060 S   0.7  1.5  24:55.22 dockerd-current                                                                                                        
118599 root      20   0 10.346g  76844  16260 S   0.7  1.9   4:41.54 etcd                                                                                                                   
    10 root      20   0       0      0      0 I   0.3  0.0   6:05.00 rcu_sched                                                                                                              
 48603 root      20   0  145440   8440   7184 S   0.3  0.2   0:01.30 sshd                                                                                                                   
118482 root      20   0  114184   4028   2752 S   0.3  0.1   0:10.08 local-up-cluste                                                                                                        
119042 root      20   0   53936  39132  28888 S   0.3  1.0   0:51.40 kube-proxy                                                                                                             
     1 root      20   0  126468   5728   3616 S   0.0  0.1   2:32.84 systemd                                                                                                                
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.33 kthreadd                                                                                                               
     3 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 rcu_gp                                                                                                                 
     4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 rcu_par_gp                                                                                                             
     6 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H-xf                                                                                                        
     8 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 mm_percpu_wq                                                                                                           
     9 root      20   0       0      0      0 S   0.0  0.0   0:10.38 ksoftirqd/0                                                                                                            
    11 root      20   0       0      0      0 I   0.0  0.0   0:00.00 rcu_bh                                                                                                                 
    12 root      rt   0       0      0      0 S   0.0  0.0   0:01.69 migration/0                                                                                                            
    13 root      rt   0       0      0      0 S   0.0  0.0   0:00.81 watchdog/0                                                                                                             
    14 root      20   0       0      0      0 S   0.0  0.0   0:00.00 cpuhp/0                                                                                                                
    15 root      20   0       0      0      0 S   0.0  0.0   0:00.00 cpuhp/1                                                                                                                
    16 root      rt   0       0      0      0 S   0.0  0.0   0:00.92 watchdog/1                                                                                                             
    17 root      rt   0       0      0      0 S   0.0  0.0   0:01.62 migration/1                                                                                                            
    18 root      20   0       0      0      0 S   0.0  0.0   0:14.91 ksoftirqd/1                                                                                                            
    20 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/1:0H-xf                                                                                                        
    21 root      20   0       0      0      0 S   0.0  0.0   0:00.00 cpuhp/2                                                                                                                
    22 root      rt   0       0      0      0 S   0.0  0.0   0:00.84 watchdog/2                                                                                                             
    23 root      rt   0       0      0      0 S   0.0  0.0   0:01.68 migration/2                                                                                                            
    24 root      20   0       0      0      0 S   0.0  0.0   0:10.84 ksoftirqd/2                                                                                                            
    26 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/2:0H-kb
```
cpu跑了100%
```
# cat container/cpu.cfs_quota_us
-1
```
查看cpu quota限制
```
cat container/cpu.cfs_period_us
100000
```
查看cpu period限制
- 修改quota限制
```
# echo 20000 container/cpu.cfs_quota_us
20000 container/cpu.cfs_quota_us
```
它意味着在每 100 ms 的时间里，被该控制组限制的进程只能使用 20 ms 的 CPU 时间，也就是说这个进程只能使用到 20% 的 CPU 带宽
```
# echo 68322 > container/tasks
```
将死循环加入控制组，cpu就降下来了

# 除了cpu子系统，cgroup还有很多其他的资源限制能力
- blkio， 为块设备限制I/O
- cpuset, 为进程分配单独的cpu核和对应的内存节点
- memory, 为进程设定内存使用的限制

# docker的Cgroup玩法
在每个子系统下面，为每个容器创建一个控制组(就是创建一个新目录)，然后将启动的容器进程加入到对应控制组的tasks文件就可以了
docker容器资源的限制就是在docker run启动是参数指定的
- 使用示例
启动cpu限制的容器
```
# docker run -it --cpu-period=100000 --cpu-quota=20000 busybox /bin/sh
```
然后去cgroup目录看一下
```
$ cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_period_us
100000
$ cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_quota_us
20000
```
笔者的环境竟然没有生成这个目录，难道是装了k8s的原因
研究了一下发现路径跟张磊的不一样
```
# ll
总用量 0
-rw-r--r--.   1 root root 0 10月  3 16:10 cgroup.clone_children
-rw-r--r--.   1 root root 0 10月  3 16:10 cgroup.procs
-r--r--r--.   1 root root 0 10月  3 16:10 cgroup.sane_behavior
drwxr-xr-x.   2 root root 0 10月  3 16:11 container
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.stat
-rw-r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage_all
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage_percpu
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage_percpu_sys
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage_percpu_user
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage_sys
-r--r--r--.   1 root root 0 10月  3 16:10 cpuacct.usage_user
-rw-r--r--.   1 root root 0 10月  3 16:10 cpu.cfs_period_us
-rw-r--r--.   1 root root 0 10月  3 16:10 cpu.cfs_quota_us
-rw-r--r--.   1 root root 0 10月  3 16:10 cpu.rt_period_us
-rw-r--r--.   1 root root 0 10月  3 16:10 cpu.rt_runtime_us
-rw-r--r--.   1 root root 0 10月  3 16:10 cpu.shares
-r--r--r--.   1 root root 0 10月  3 16:10 cpu.stat
drwxr-xr-x.   4 root root 0 9月  29 21:30 kubepods.slice
drwxr-xr-x.   2 root root 0 9月  29 21:30 kube-proxy
-rw-r--r--.   1 root root 0 10月  3 16:10 notify_on_release
-rw-r--r--.   1 root root 0 10月  3 16:10 release_agent
drwxr-xr-x. 114 root root 0 9月  29 12:55 system.slice
-rw-r--r--.   1 root root 0 10月  3 16:10 tasks
drwxr-xr-x.   4 root root 0 9月  29 12:55 user.slice
```
在system.slice目录下面
```
ll system.slice/*4c46475c05c0*
system.slice/docker-4c46475c05c0372ff550718235cb02427be6afff3e658112a83e69817c28f74b.scope:
总用量 0
-rw-r--r--. 1 root root 0 10月  3 17:04 cgroup.clone_children
-rw-r--r--. 1 root root 0 10月  3 16:35 cgroup.procs
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.stat
-rw-r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_all
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_percpu
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_percpu_sys
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_percpu_user
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_sys
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_user
-rw-r--r--. 1 root root 0 10月  3 16:35 cpu.cfs_period_us
-rw-r--r--. 1 root root 0 10月  3 16:35 cpu.cfs_quota_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.rt_period_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.rt_runtime_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.shares
-r--r--r--. 1 root root 0 10月  3 17:04 cpu.stat
-rw-r--r--. 1 root root 0 10月  3 17:04 notify_on_release
-rw-r--r--. 1 root root 0 10月  3 17:04 tasks

system.slice/var-lib-docker-containers-4c46475c05c0372ff550718235cb02427be6afff3e658112a83e69817c28f74b-shm.mount:
总用量 0
-rw-r--r--. 1 root root 0 10月  3 17:04 cgroup.clone_children
-rw-r--r--. 1 root root 0 10月  3 17:04 cgroup.procs
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.stat
-rw-r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_all
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_percpu
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_percpu_sys
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_percpu_user
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_sys
-r--r--r--. 1 root root 0 10月  3 17:04 cpuacct.usage_user
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.cfs_period_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.cfs_quota_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.rt_period_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.rt_runtime_us
-rw-r--r--. 1 root root 0 10月  3 17:04 cpu.shares
-r--r--r--. 1 root root 0 10月  3 17:04 cpu.stat
-rw-r--r--. 1 root root 0 10月  3 17:04 notify_on_release
-rw-r--r--. 1 root root 0 10月  3 17:04 tasks
```

# top和/proc的问题
张磊所说的容器中可以看到宿主机的/proc的问题在笔者的环境里面没有出现
```
/ # top
Mem: 3907244K used, 108948K free, 45876K shrd, 8K buff, 920448K cached
CPU: 30.7% usr  2.5% sys  0.0% nic 66.6% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 2.41 2.07 2.11 3/616 7
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
    1     0 root     S     1256  0.0   1  0.0 /bin/sh
    7     1 root     R     1248  0.0   0  0.0 top
```
容器中的top命令不能看到宿主机的/proc内容,不懂是否有修改
课后回答有人说使用lxcfs把/var/lib/lxcfs/proc/\*挂载到容器的/proc/\*目录中
