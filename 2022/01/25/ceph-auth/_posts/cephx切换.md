---
title: cephx切换
date: 2020-10-15 21:07:28
tags: Ceph
---

## Cephx在线关闭问题
- 1.关闭cephx需要重启啥操作
(1) 修改配置文件
```
auth cluster required = none
auth service required = none
auth client required = none
```

(2) 重启服务
先重启mon，在重启osd

- 2.存量问题
2.1 新的客户端没有问题
2.2 已经打开的客户端会有问题
读写会卡主，现象,原因分析
（1）客户端跟mon的conn会mark down，建立新的连接，然后auth认证返回95错误
（2）客户端跟osd的conn不会mark down，会报address错误，因为原来osd的进程id和现在的进程id不一样，所以一直在重试
ps：
1.正常的情况（不切cephx）osd重启为什么没有错误，正常的osd重启也一直有osd的进程id和现在的进程id，但是osd的重启会更新osdmap，mon会给客户端发送osdmap（客户端订阅了），在客户端处理handle_osdmap的流程里面有对address改变的处理，会close_session，然后会mark down连接
而在改了cephx之后，mon已经不能发送osdmap给客户端了

2.mon会什么conn会重建呢，因为monclient客户端有一个tick操作
```
void MonClient::_reopen_session(int rank)
{
  assert(monc_lock.is_locked());
  ldout(cct, 10) << __func__ << " rank " << rank << dendl;

  active_con.reset();
  pending_cons.clear();

  _start_hunting();
}
```
pending_cons.clear()这个会释放掉对象
```
MonConnection::~MonConnection()
{
  if (con) {
    con->mark_down();
    con.reset();
  }
}
```
这个会调用mark_down
