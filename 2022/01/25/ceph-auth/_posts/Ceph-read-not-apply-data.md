---
title: 数据还没有apply前进行读
date: 2021-02-23 17:41:32
tags: Ceph
---

# 问题
ceph在写完日志之后就回调commit发送ack，write操作就完成了，这个时候表示数据写完成，但是
还是不能读的，只有apply回调完成才变成可读，那这个可读是哪里的控制的呢？

# 分析
## 1. 对象上面有读写锁
- 在写的时候会进行ondiskwritelock
```
void ondisk_read_lock() {
   lock.Lock();
   readers_waiting++;
   while (unstable_writes)
     cond.Wait(lock);
   readers_waiting--;
   readers++;
   lock.Unlock();
 }
```
unstable_writes增加，也就是对一个对象来写请求的时候会增加unstable_write在看一下读请求

- 在execute_ctx里面有如下代码
```
void PrimaryLogPG::execute_ctx(OpContext *ctx)
{
  ....
  if (op->may_read()) {
    dout(10) << " taking ondisk_read_lock" << dendl;
    obc->ondisk_read_lock();
  }
  ....
}
```
如果是读请求会申请ondisk_read_lock
```
void ondisk_read_lock() {
   lock.Lock();
   readers_waiting++;
   while (unstable_writes)
     cond.Wait(lock);
   readers_waiting--;
   readers++;
   lock.Unlock();
 }
```
这个ondisk_read_lock会在unstable_writes不为0的时候进行条件变量的阻塞，

- 而unstable_writes减少是在issue_repop的这个回调中
```
void PrimaryLogPG::issue_repop(RepGather *repop, OpContext *ctx)
{
  ...
  Context *on_all_commit = new C_OSD_RepopCommit(this, repop);
 Context *on_all_applied = new C_OSD_RepopApplied(this, repop);
 Context *onapplied_sync = new C_OSD_OndiskWriteUnlock(
   ctx->obc,
   ctx->clone_obc,
   unlock_snapset_obc ? ctx->snapset_obc : ObjectContextRef());
  ...
}
```
C_OSD_OndiskWriteUnlock中回调进行ondisk_write_unlock
```
class PrimaryLogPG::C_OSD_OndiskWriteUnlock : public Context {
  ObjectContextRef obc, obc2, obc3;
  public:
  C_OSD_OndiskWriteUnlock(
    ObjectContextRef o,
    ObjectContextRef o2 = ObjectContextRef(),
    ObjectContextRef o3 = ObjectContextRef()) : obc(o), obc2(o2), obc3(o3) {}
  void finish(int r) override {
    obc->ondisk_write_unlock();
    if (obc2)
      obc2->ondisk_write_unlock();
    if (obc3)
      obc3->ondisk_write_unlock();
  }
};

void ondisk_write_unlock() {
  lock.Lock();
  assert(unstable_writes > 0);
  unstable_writes--;
  if (!unstable_writes && readers_waiting)
    cond.Signal();
  lock.Unlock();
}
```
