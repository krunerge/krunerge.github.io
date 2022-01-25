---
title: Rbd快照数据一致性浅析
date: 2020-07-11 19:35:25
tags: Ceph rbd
---

# 导语
快照一般是指数据存储的某一时刻的状态记录，类似于给数据按下快门拍了一张照片，所以也叫snapshot。而存储系统的快照在云计算中广泛使用，比如块存储的快照。很多其他高级功能基本都要依赖快照来实现，比如备份、热迁移等。而对于快照，我们经常会问的一个问题就是快照的数据是不是完整的，会不会出现快照回滚之后数据丢失。其实这也就是我们常说的快照数据一致性问题。

下面主要分以下几点进行讨论:
(1) 一致性的分类
(2) Ceph中一致性的实现

下面开始介绍

# 一致性分类
快照这里我们主要是讲用在虚拟机块存储上的快照，首先看一下下面张图，
![1](1.png)
从上面的图可以看出，我们的数据会进过应用层、文件系统层最后到达块设备层。每个层次可能会有一部分缓存，比如应用层里面的程序会有读写缓存，文件系统层会有page cache，块设备层有块设备的缓存。
根据这三层一致性主要分为以下几种：
（1）奔溃一致性快照
奔溃一致性其实没有做特殊的保障，这时候快照存储的数据就相当于虚拟机突然掉电时候块设备上存储的数据状态，对于我们云计算中的块存储可能上图中的三个层中的缓存脏数据都没有刷到块设备。

（2）文件系统一致性快照
文件系统一致性快照是在做快照前，文件系统被暂时冻结，文件系统层的缓存脏数据刷到块设备中。冻结用于拒绝用户层应用的IO请求。

（3）应用一致性快照
应用一致性快照是在做快照前，应用被暂时冻结，并把应用层缓存的脏数据刷到块存储
从上面三种快照一致性的分离中我们发现我们没有对块设备的缓存持久化进行归类，其实根据不同的存储系统有些可以归到奔溃一致性快照里面。
这里我们不细介绍应用层和文件系统层了，主要极少一下存储系统块设备层的数据快照一致性做法，下面我们来看看Ceph中的rbd块设备是如何维护着一致性的。

# Rbd的快照
在做块存储快照的时候，我们最希望的就是rbd快速没有io在过来、内部飞行的io都出来回调完成、rbd缓存中的脏数据都已经刷到磁盘上，那这时候做快照，无论使用什么姿势，数据肯定是完整没问题的。我们看一下rbd快照前做了什么，如下是12.2.10 L版的代码
```
  *            <start>
  *               |
  *               v
  *           STATE_SUSPEND_REQUESTS
  *               |
  *               v
  *           STATE_SUSPEND_AIO * * * * * * * * * * * * *
  *               |                                     *
  *               v                                     *
  *           STATE_APPEND_OP_EVENT (skip if journal    *
  *               |                  disabled)          *
  *   (retry)     v                                     *
  *   . . . > STATE_ALLOCATE_SNAP_ID                    *
  *   .           |                                     *
  *   .           v                                     *
  *   . . . . STATE_CREATE_SNAP * * * * * * * * * *     *
  *               |                               *     *
  *               v                               *     *
  *           STATE_CREATE_OBJECT_MAP (skip if    *     *
  *               |                    disabled)  *     *
  *               |                               *     *
  *               |                               v     *
  *               |              STATE_RELEASE_SNAP_ID  *
  *               |                     |               *
  *               |                     v               *
  *               \----------------> <finish> < * * * * *
```
可以看到在做快照前做了两个动作suspend_requests和suspend_aio，其中suspend_requests就是挂住io，阻塞请求进来，那suspend_aio是什么呢？其实qemu在使用librbd读写数据的时候都是使用的异步接口进行读写，所以这里的aio就是在librbd中飞行的io，简而言之就是要把飞行的io运行完落盘而不是挂住，那这和我们上面希望的是差不多，让我们在仔细分析一下这个两个动作是怎么做到的。
## 1. suspend_requests阻塞请求
阻塞请求，因为librbd里面有请求队列，那简单的做法就是在出队列的时候卡主，不让io出队列运行，其实这里的io主要是对写io，读io的写法不会修改数据所以不会出现数据的不一致。
```
template <typename I>
void ImageRequestWQ<I>::block_writes(Context *on_blocked) {
  assert(m_image_ctx.owner_lock.is_locked());
  CephContext *cct = m_image_ctx.cct;

  {
    RWLock::WLocker locker(m_lock);
    ++m_write_blockers; //write blocker计数增加
    ...
    }
  }

  ...
}
```
上面的函数就是阻塞写请求，有一个m_write_blockers计数器，每block一次，计数器增加1，那这个计数器在哪里使用呢？下面这个是请求出队列的地方
```
template <typename I>
void *ImageRequestWQ<I>::_void_dequeue() {
  ...

  bool lock_required;
  bool refresh_required = m_image_ctx.state->is_refresh_required();
  {
    RWLock::RLocker locker(m_lock);
    bool write_op = peek_item->is_write_op();
    lock_required = is_lock_required(write_op);
    if (write_op) {
      if (!lock_required && m_write_blockers > 0) { // 不出队列
        // missing lock is not the write blocker
        return nullptr;
      }

      ...
    }
  }

...
}
```
上面出队列的时候对请求判断了一下，如果是写请求，并且m_write_blockers大于0，请求就不处理了，这样就对所有的写请求不出队列处理了。

## 2. suspend_aio刷飞行io
飞行io就是正在运行的io，这里我们也只需要考虑写io，这些io操作都在运行中，数据可能还没有完全落盘，看一下rbd是不是这样呢？下面还是block_write函数
```
void ImageRequestWQ<I>::block_writes(Context *on_blocked) {
  ...

  // ensure that all in-flight IO is flushed
  m_image_ctx.flush(on_blocked);
}
```
可以看到是通过image上下文的flush在加了一个on_blocked，这个on_blocked是一个条件变量
```
int ImageRequestWQ<I>::block_writes() {
  C_SaferCond cond_ctx;
  block_writes(&cond_ctx); //阻塞在这了
  return cond_ctx.wait();
}
```
也就是通过这个条件变量阻塞在这儿了，等待飞行io的运行完。我们看一下image上下文的flush待着这个条件变量做了啥？
```
void ImageCtx::flush(Context *on_safe) {
    // ensure no locks are held when flush is complete
    ...
    // 块设备缓存flush
    if (object_cacher != NULL) {
      // flush cache after completing all in-flight AIO ops
      on_safe = new C_FlushCache(this, on_safe);
    }
    // flush异步操作
    flush_async_operations(on_safe);
  }
```
可以看出这里除了等待异步飞行io的完成还根据rbd是否开启缓存进行rbd cache的脏数据下刷。rbd cache是基于多条LRU构造的，根据LRU缓存算法镜像flush，这里不多介绍，下面看一下异步操作是如何flush的。
```
void ImageCtx::flush_async_operations(Context *on_finish) {
    {
      Mutex::Locker l(async_ops_lock);
      if (!async_ops.empty()) {
        ldout(cct, 20) << "flush async operations: " << on_finish << " "
                       << "count=" << async_ops.size() << dendl;
        async_ops.front()->add_flush_context(on_finish);
        return;
      }
    }
    on_finish->complete(0);
  }
```
这里的async_ops一个异步操作的链表，关键是下面这一行
```
async_ops.front()->add_flush_context(on_finish);
```
在异步操作链表的第一个异步请求上面增加了一个on_finish，这个就是我们之前条件变量回调的触发，这个一回调，上面卡住的地方就可以继续往下运行。我用下图示意
![2](2.png)
各个异步操作完成时间各不相同，那看一下异步操作链表的操作
![3](3.png)
从上面示意图可以看出，on_finish一直挂在链表第一个异步操作上面，知道所有的异步操作完成，会触发回调，解除卡住，也就达到了flush异步操作的目的。

这样上面就已经介绍了block写请求、flush飞行io、flush rbd缓存。在这种状态下，我们怎么做快照都是数据完整的，但是这个完整还是只针对块设备，还没有包括应用层和文件系统层。Ceph有对这种进行处理吗？

在最近的提交中我看到了这个commit
```
librbd: API for quiesce callbacks
```
增加了静默的回调，什么意识？再看一下做快照的导图
```
*            <start>
*               |
*               v
*           STATE_NOTIFY_QUIESCE
*               |
*               v
*           STATE_SUSPEND_REQUESTS
*               |
*               v
*           STATE_SUSPEND_AIO * * * * * * * * * * * * * * *
*               |                                         *
*               v                                         *
*           STATE_APPEND_OP_EVENT (skip if journal        *
*               |                  disabled)              *
*   (retry)     v                                         *
*   . . . > STATE_ALLOCATE_SNAP_ID                        *
*   .           |                                         *
*   .           v                                         *
*   . . . . STATE_CREATE_SNAP * * * * * * * * * * *       *
*               |                                 *       *
*               v                                 *       *
*           STATE_CREATE_OBJECT_MAP (skip if      *       *
*               |                    disabled)    *       *
*               v                                 *       *
*           STATE_CREATE_IMAGE_STATE (skip if     *       *
*               |                     not mirror  *       *
*               |                     snapshot)   *       *
*               |                                 v       *
*               |              STATE_RELEASE_SNAP_ID      *
*               |                     |                   *
*               |                     v                   *
*               \------------> STATE_NOTIFY_UNQUIESCE < * *
*                                     |
*                                     v
*                                  <finish>
```
可以看到增加了notify_quiesce,通知静默，看了一下代码其实就是用户可以提前注册好一个回调，在做快照的时候可以触发这个回调，嗯，这个回调做什么用呢，聪明的你想想，对啊，就是可以做应用层和文件系统层的相关一致性操作，可以通过回调触发一些命令行或者脚本来做一些在块设备上层的刷数据。Good！！！

到这里大致就介绍完了快照一致性，大家可以再花几秒钟回顾一下。

# 参考资料
https://blog.csdn.net/zhouxukun123/article/details/75093978
