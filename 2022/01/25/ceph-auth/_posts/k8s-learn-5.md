---
title: k8s learn(5) 从进程说起:容器到底是怎么一回事
date: 2018-10-03 14:38:31
tags: k8s 张磊
---

# 1.程序跟进程
- 程序是静态的
- 进程是动态的

# 2.容器是一种进程
```
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL)
```
通过Cgroups 技术是用来制造约束的主要手段， 而Namespace 技术则是用来修改进程视图的主要方法
提供6种命令空间：
```
PID
Mount
UTS
IPC
Network
User
```
# 3.容器中的进程跟宿主机中进程的对应关系
- 在容器中，容器的启动进程在容器里面是1号进程（假init进程）
```
# docker run -it busybox /bin/sh
```
容器中进程
```
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
   36 root      0:00 ps
```
- 在宿主机中，如何查看容器的进程呢，进程名其实就是容器的启动进程，比如上面的/bin/sh，通过docker inspect查看容器进程
```
docker inspect xxxx
```
从中找到status pid就是容器的进程id
我们可以在容器中运行一个top命令，看一下进程的关系
## 在容器中
top命令是init的子进程
## 在宿主机上
```
# pstree -p 2837
dockerd-current(2837)─┬─docker-containe(105880)─┬─docker-containe(48506)─┬─sh(48521)───top(50346)
                      │                         │                        ├─{docker-containe}(48507)
                      │                         │                        ├─{docker-containe}(48508)
                      │                         │                        ├─{docker-containe}(48509)
                      │                         │                        ├─{docker-containe}(48510)
                      │                         │                        ├─{docker-containe}(48511)
                      │                         │                        ├─{docker-containe}(48513)
                      │                         │                        └─{docker-containe}(48545)
                      │                         ├─docker-containe(119459)─┬─pause(119475)
                      │                         │                         ├─{docker-containe}(119460)
                      │                         │                         ├─{docker-containe}(119461)
                      │                         │                         ├─{docker-containe}(119462)
                      │                         │                         ├─{docker-containe}(119463)
                      │                         │                         ├─{docker-containe}(119464)
                      │                         │                         ├─{docker-containe}(119465)
                      │                         │                         ├─{docker-containe}(119466)
                      │                         │                         └─{docker-containe}(119509)
                      │                         ├─docker-containe(119525)─┬─kube-dns(119546)─┬─{kube-dns}(119567)
                      │                         │                         │                  ├─{kube-dns}(119568)
                      │                         │                         │                  ├─{kube-dns}(119569)
                      │                         │                         │                  ├─{kube-dns}(119570)
                      │                         │                         │                  ├─{kube-dns}(119571)
                      │                         │                         │                  ├─{kube-dns}(119572)
                      │                         │                         │                  ├─{kube-dns}(119573)
                      │                         │                         │                  ├─{kube-dns}(119583)
                      │                         │                         │                  ├─{kube-dns}(11921)
                      │                         │                         │                  └─{kube-dns}(14741)
                      │                         │                         ├─{docker-containe}(119526)
                      │                         │                         ├─{docker-containe}(119527)
                      │                         │                         ├─{docker-containe}(119528)
                      │                         │                         ├─{docker-containe}(119529)
                      │                         │                         ├─{docker-containe}(119530)
                      │                         │                         ├─{docker-containe}(119531)
                      │                         │                         ├─{docker-containe}(119532)
                      │                         │                         ├─{docker-containe}(119533)
                      │                         │                         └─{docker-containe}(119534)
                      │                         ├─docker-containe(119585)─┬─dnsmasq-nanny(119602)─┬─dnsmasq(119638)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119623)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119624)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119625)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119626)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119634)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119636)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119788)
                      │                         │                         │                       ├─{dnsmasq-nanny}(119787)
                      │                         │                         │                       └─{dnsmasq-nanny}(49454)
                      │                         │                         ├─{docker-containe}(119586)
                      │                         │                         ├─{docker-containe}(119587)
                      │                         │                         ├─{docker-containe}(119588)
                      │                         │                         ├─{docker-containe}(119589)
                      │                         │                         ├─{docker-containe}(119590)
                      │                         │                         ├─{docker-containe}(119591)
                      │                         │                         ├─{docker-containe}(119592)
                      │                         │                         └─{docker-containe}(119596)
                      │                         ├─docker-containe(119639)─┬─sidecar(119656)─┬─{sidecar}(119676)
                      │                         │                         │                 ├─{sidecar}(119677)
                      │                         │                         │                 ├─{sidecar}(119678)
                      │                         │                         │                 ├─{sidecar}(119679)
                      │                         │                         │                 ├─{sidecar}(119689)
                      │                         │                         │                 ├─{sidecar}(119690)
                      │                         │                         │                 ├─{sidecar}(119691)
                      │                         │                         │                 ├─{sidecar}(125464)
                      │                         │                         │                 └─{sidecar}(33842)
                      │                         │                         ├─{docker-containe}(119640)
                      │                         │                         ├─{docker-containe}(119641)
                      │                         │                         ├─{docker-containe}(119642)
                      │                         │                         ├─{docker-containe}(119643)
                      │                         │                         ├─{docker-containe}(119644)
                      │                         │                         ├─{docker-containe}(119645)
                      │                         │                         ├─{docker-containe}(119646)
                      │                         │                         └─{docker-containe}(119647)
                      │                         ├─{docker-containe}(105881)
                      │                         ├─{docker-containe}(105882)
                      │                         ├─{docker-containe}(105883)
                      │                         ├─{docker-containe}(105884)
                      │                         ├─{docker-containe}(105885)
                      │                         ├─{docker-containe}(105886)
                      │                         ├─{docker-containe}(105887)
                      │                         ├─{docker-containe}(105895)
                      │                         ├─{docker-containe}(106063)
                      │                         ├─{docker-containe}(112355)
                      │                         ├─{docker-containe}(112412)
                      │                         ├─{docker-containe}(112467)
                      │                         ├─{docker-containe}(112987)
                      │                         ├─{docker-containe}(113989)
                      │                         ├─{docker-containe}(114932)
                      │                         └─{docker-containe}(6250)
                      ├─docker-proxy-cu(3090)─┬─{docker-proxy-cu}(3091)
                      │                       ├─{docker-proxy-cu}(3092)
                      │                       ├─{docker-proxy-cu}(3093)
                      │                       ├─{docker-proxy-cu}(3094)
                      │                       ├─{docker-proxy-cu}(3095)
                      │                       └─{docker-proxy-cu}(3619)
                      ├─{dockerd-current}(2839)
                      ├─{dockerd-current}(2840)
                      ├─{dockerd-current}(2841)
                      ├─{dockerd-current}(2845)
                      ├─{dockerd-current}(2850)
                      ├─{dockerd-current}(2851)
                      ├─{dockerd-current}(2860)
                      ├─{dockerd-current}(2861)
                      ├─{dockerd-current}(2864)
                      ├─{dockerd-current}(2865)
                      ├─{dockerd-current}(3012)
                      ├─{dockerd-current}(3027)
                      ├─{dockerd-current}(3028)
                      ├─{dockerd-current}(3029)
                      ├─{dockerd-current}(3032)
                      ├─{dockerd-current}(3082)
                      ├─{dockerd-current}(3096)
                      ├─{dockerd-current}(3098)
                      ├─{dockerd-current}(3099)
                      ├─{dockerd-current}(3101)
                      ├─{dockerd-current}(3102)
                      └─{dockerd-current}(3107)
```
其中，笔者的容器进程关系是这条
```
dockerd-current(2837)─┬─docker-containe(105880)─┬─docker-containe(48506)─┬─sh(48521)───top(50346)
```
其中2837是dockerd驻留进程，其父进程是1（真正的init进程）,进程105880和48506也是docker相关的进程，不是容器进程，
48521是容器进程，50346是top进程

从这可以看出容器里和宿主机上面的进程关系
