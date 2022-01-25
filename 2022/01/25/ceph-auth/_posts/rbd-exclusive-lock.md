---
title: rbd请求是互斥的？
date: 2020-03-15 14:48:31
tags: rbd
---

# 问题
最近在分析rbd快照的时候，发现需要申请exclusive_lock，不止快照，基本上很多rbd操作都要求获取这个锁，那这个锁是怎么运行的呢，是不是只能有一个客户端可以获取到这个锁，然后进行rbd卷的操作？如果是这样那qemu虚拟机在写数据的时候的，nova-compute就根本不能操作这个卷了，因为qemu和nova-compute在rbd看来就是他的两个客户端，他们都要求申请互斥锁，这显然和我们的实验不同

# 分析
以分析快照创建代码为例看rbd请求
```
template <typename I>
void Operations<I>::snap_create(const cls::rbd::SnapshotNamespace &snap_namespace,
				const std::string& snap_name,
				Context *on_finish) {
  CephContext *cct = m_image_ctx.cct;
  ldout(cct, 5) << this << " " << __func__ << ": snap_name=" << snap_name
                << dendl;

  if (m_image_ctx.read_only) {
    on_finish->complete(-EROFS);
    return;
  }

  m_image_ctx.snap_lock.get_read();
  if (m_image_ctx.get_snap_id(snap_namespace, snap_name) != CEPH_NOSNAP) {
    m_image_ctx.snap_lock.put_read();
    on_finish->complete(-EEXIST);
    return;
  }
  m_image_ctx.snap_lock.put_read();

  C_InvokeAsyncRequest<I> *req = new C_InvokeAsyncRequest<I>(
    m_image_ctx, "snap_create", true,
    boost::bind(&Operations<I>::execute_snap_create, this, snap_namespace, snap_name,
		_1, 0, false),
    boost::bind(&ImageWatcher<I>::notify_snap_create, m_image_ctx.image_watcher,
                snap_namespace, snap_name, _1),
    {-EEXIST}, on_finish);
  req->send();
}
```
跟着librbd的接口我们会一路走到这里，在上面的函数中我们看到C_InvokeAsyncRequest请求中传了两个处理函数，execute_snap_create和notify_snap_create，为什么会传两个函数，难道。。。。，没错就是你想的那样，首先，这里要说一下，
```
1.rbd的exclusive_lock既然是互斥锁，那它的拥有者也就只有一个“人”，我们称他为owner
2.另一方面，虽然这是个互斥锁，但是很多客户端可以同时“拥有”这个互斥锁进行rbd请求，（ps.我会告诉你我在拥有上面加了引号），这个我们平台在操作rbd的时候也知道这现象
```
再说说这两个请求：
- 1.如果当前的客户端是exclusive_lock的拥有者，（这里我没有加引号，是真正唯一的拥有者），就执行execute_snap_create,也就是当前客户端区创建快照，创建快照主要是创建一些元数据，这些元数据主要用于管理使用
- 2.如果当前的客户端不是exclusive_lock的拥有者，就执行notify_snap_create，看函数的名字notify，你猜notify谁，是的是notify exclusive_lock拥有者，让拥有这个互斥锁的客户端去运行创建快照操作，这样委托可以吗？可以反正创建快照，只是创建一些管理元数据，谁创建不是创建，nice

那是不是这样呢，课代表来了
```
template <typename I>
struct C_InvokeAsyncRequest : public Context {
  /**
   * @verbatim
   *
   *               <start>
   *                  |
   *    . . . . . .   |   . . . . . . . . . . . . . . . . . .
   *    .         .   |   .                                 .
   *    .         v   v   v                                 .
   *    .       REFRESH_IMAGE (skip if not needed)          .
   *    .             |                                     .
   *    .             v                                     .
   *    .       ACQUIRE_LOCK (skip if exclusive lock        .
   *    .             |       disabled or has lock)         .
   *    .             |                                     .
   *    .   /--------/ \--------\   . . . . . . . . . . . . .
   *    .   |                   |   .
   *    .   v                   v   .
   *  LOCAL_REQUEST       REMOTE_REQUEST
   *        |                   |
   *        |                   |
   *        \--------\ /--------/
   *                  |
   *                  v
   *              <finish>
   *
   * @endverbatim
   */
```
在C_InvokeAsyncRequest请求中有这样的注释，local_request 和 remote_request更加生动形象，至于具体流程的跟踪，可以使用两个python交互式客户端操作一个rbd，然后观察两个客户端的日志，这里就不多说了。
