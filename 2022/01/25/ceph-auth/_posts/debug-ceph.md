---
title: debug ceph
date: 2020-05-13 20:10:48
tags:
---

# centos安装clion
- 1.安装jdk
```
yum install java-1.8.0-openjdk
```

- 2.DISPLAY
```
export DISPLAY=:0
```

- 3.X11

```
vi /etc/ssh/sshd_config
    配置：X11Forwarding yes
    然后重启服务service sshd restart

然后确保xshell客户端配置为：
    属性-连接-SSH-隧道：
    X11转移-（选中）转发X11连接到-（选中）Xmanager

然后打开xshell会话后：
    echo $DISPLAY 查看是有值的
    此时直接运行脚本可以打开程序GUI界面
```
