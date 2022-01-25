---
title: recover mysql cluster
date: 2020-03-05 20:22:16
tags: mysql
---

# mysql galera集群恢复
```
# 正常重启流程

1、先关闭所有集群的mysqld

systemctl stop mysqld
systemctl stop mariadb

2、在master上使用

sudo -u mysql /usr/libexec/mysqld --wsrep-cluster-address='gcomm://' &
show status like "wsrep%";

3、成功后在slave上正常启动mysql

systemctl restart mariadb
4、所有slave启动成功之后在master上操作：

ps -ef|grep mysqld
kill -9 pid
systemctl restart mariadb
```
