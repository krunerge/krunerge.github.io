---
title: k8s learn(8)重新认识Docker容器
date: 2018-10-08 22:21:46
tags: k8s 张磊
---

# 镜像上传到Docker hub
- 1.docker login登陆
- 2.docker tag  hellword krunerge/hellword:v1
其中krunerge是你的仓库名，也是你docker hub的用户名
- 3.docker push krunerge/hellword:v1

# docker exec原理
- 1.找到容器的进程id
```
# docker inspect --format '{{.State.Pid}}' e4065bd03082
48987
```
- 2.找到进程的相关的namespace
```
# ll /proc/48987/ns
总用量 0
lrwxrwxrwx. 1 root root 0 10月  8 22:19 cgroup -> cgroup:[4026531835]
lrwxrwxrwx. 1 root root 0 10月  8 22:19 ipc -> ipc:[4026533013]
lrwxrwxrwx. 1 root root 0 10月  8 22:05 mnt -> mnt:[4026533009]
lrwxrwxrwx. 1 root root 0 10月  8 22:05 net -> net:[4026533016]
lrwxrwxrwx. 1 root root 0 10月  8 22:19 pid -> pid:[4026533014]
lrwxrwxrwx. 1 root root 0 10月  8 22:19 pid_for_children -> pid:[4026533014]
lrwxrwxrwx. 1 root root 0 10月  8 22:19 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 10月  8 22:19 uts -> uts:[4026533012]
```
- 3.docker exec的原理就是创建一个新进程，加入到之前容器进程的namespace中，就是通过上面找到的namespace
使用setns()系统调用实现，模拟实现如下：
```
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do{ perror(msg); exit(EXIT_FAILURE);} while(0)

int main(int argc, char *argv[]){
  int fd;

  fd = open(argv[1], O_RDONLY);
  if (setns(fd, 0) == -1){
      errExit("setns");
  }
  execvp(argv[2], &argv[2]);
  errExit("execvp");
}
```
# docker的数据卷
- docker支持两种数据卷声明方式
```
1. docker run -v /test ...
这个会创建一个临时目录/var/lib/docker/volumes/[VOLUME_ID]/_data
2. docker run -v /home:/test ...
使用宿主机的/home目录挂载
```
- 数据挂载时机
1.首先使用镜像生成rootfs
2.然后挂载数据卷到容器目录（使用的绑定挂载bind mount）
3.在执行chroot操作
- 数据卷是挂载容器的可读写层
docker commit的时候不会提交，因为docker commit是在宿主机执行，看不到，但是commit之后会有一个空挂载点目录生成
