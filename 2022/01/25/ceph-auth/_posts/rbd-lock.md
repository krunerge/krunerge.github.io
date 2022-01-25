---
title: 使用rbd来实现分布式锁
date: 2019-01-12 19:37:39
tags: openstack nova ceph
---

# 问题
————————————————————————————————————————————————————————————————————————————
最近在做虚拟机的在线快照，使用的是ceph后端存储，快照使用的是对虚拟机的所有盘进行snap+clone，处于对ceph性能的考虑，准备半夜对ceph进行flatten操作，同时限制同一时刻只有一个rbd卷在做flatten操作，这就涉及到多个计算节点在做flatten的时候如何保证ceph集群只有一个clone的卷在做flatten，由于，flatten都是计算节点的nova-compute定时触发的

# 解决方法
————————————————————————————————————————————————————————————————————————————
可以使用rbd卷的排他锁实现，ceph的rbd提供了两个接口
- 获取锁
```python
lock_acquire(self, lock_mode)
Acquire a managed lock on the image.

Parameters:	lock_mode (int) – lock mode to set
Raises:	ImageBusy if the lock could not be acquired
```
- 释放锁
```python
lock_release(self)
Release a managed lock on the image that was previously acquired.
```
同时在ceph的某一个池中生成一个rbd，作为rbd锁，类似文件锁
大致思想是每一个nova-compute在准备启动flatten操作前都去rbd锁申请一下排他锁，只有第一个申请的才能申请到，其他nova-compute要执行flatten时，都会申请不到报出错，这样就保证了nova-compute并发flatten，导致ceph压力过大

# 具体实现
———————————————————————————————————————————————————————————————————————————
为了实现上锁，解锁，以及异常也能释放锁，决定封装一个rbdlock类，使用python的with操作
实现大致如下：
```python
class RbdLock(object):
    def __init__(self, pool, name, driver):
        self.pool = pool
        self.name = name
        self.driver = driver
        self.lock_time = None
        self.volume = None
        self.ioctx = None
        self.client = None

    def __enter__(self):
        self.lock()
        return self

    def lock(self):
        if not self.driver.lock_rbd_exist(CONF.libvirt.portrait_image_pool, "flattening_lock"):
            LOG.info("periodic: create lock rbd use to lock flattening opertion")
            self.driver.create_lock_rbd(CONF.libvirt.portrait_image_pool, "flattening_lock")

        start_time = time.time()
        self.client = rados.Rados(rados_id=CONF.libvirt.rbd_user,
                                  conffile=CONF.libvirt.images_rbd_ceph_conf)
        try:
            self.client.connect()
            self.ioctx = self.client.open_ioctx(self.pool)
        except rados.Error:
            # shutdown cannot raise an exception
            self.client.shutdown()
            raise
        try:
            self.volume = rbd.Image(self.ioctx, self.name)
        except rbd.ImageNotFound:
            with excutils.save_and_reraise_exception():
                LOG.debug("rbd image %s does not exist", self.name)
                self.disconnect_from_rados()
        except rbd.Error:
            with excutils.save_and_reraise_exception():
                LOG.exception("error opening rbd image %s" % self.name)
                self.disconnect_from_rados()

        try:
            self.trylock()
        except Exception:
            try:
                self.volume.close()
            finally:
                self.disconnect_from_rados()
                raise

        self.lock_time = time.time()
        LOG.info('Acquired rbd lock "%s/%s" after waiting %0.3fs', self.pool, self.name,
                 (self.lock_time - start_time))

    def unlock(self):
        LOG.info("unlock rbd lock...")
        if self.lock_time is None:
            raise Exception('Unable to release an unacquired "%s/%s" lock' % (self.pool, self.name))

        try:
            unlock_time = time.time()
            LOG.debug('Releasing rbd lock "%s/%s" after holding it for %0.3fs',
                      self.pool, self.name, (unlock_time - self.lock_time))
            self.tryunlock()
            self.lock_time = None
        except Exception:
            LOG.exception('Could not unlock the acquired lock "%s/%s"' % (self.pool, self.name))
        finally:
            try:
                self.volume.close()
            finally:
                self.disconnect_from_rados()

    def trylock(self):
        self.volume.lock_acquire(0)

    def tryunlock(self):
        self.volume.lock_release()

    def __exit__(self, exc_type, exc_value, exc_tb):
        self.unlock()

    def disconnect_from_rados(self):
        LOG.info('disconnect from "%s/%s"' % (self.pool, self.name))
        self.ioctx.close()
        self.client.shutdown()
```
