---
title: 你有遇到在创建rbd快照的时候虚拟机block io挂掉
date: 2020-03-15 19:28:31
tags: rbd openstack
---

# 问题
最近遇到一个问题，我们平台有定时快照功能，快照的个数是有限制的，比如一个虚拟机同一时刻最多有5个快照，如果要创建第六个快照，需要先把时间最久的第一个快照删除，然后再创建新快照，我们使用的是Openstack+Ceph的标准云平台，其中ceph的版本是10.2.7的J版。在创建快照的时候出现虚拟机挂掉，虚拟机qemu日志如下：
```
block i/o error in device 'drive-virtio-disk0': Cannot send after transport endpoint shutdown (108)
```
而且出现了大概一分多钟的慢请求告警

# 分析
- 1.我们的猜想
- （1）删除快照的压力
首先我们知道rbd的快照创建的时候是秒级的，因为只是创建了一些元数据，不会对ceph集群产生很多的压力，但是删除快照不一样，删除的快照虽然是异步操作，但是会产生数据的删除操作，而且是删除最老的快照，所以快照的数据很多已经经过cow拷贝了数据，所以删除快照会产生很大的存储压力

- （2）osd所在磁盘性能比较差
我们也注意到了这套出问题的ceph集群在使用ceph osd perf发现这套集群osd磁盘的性能参差不齐，有些延迟有1000-2000ms，可以这是个根源，那为什么会出现虚拟机挂掉呢？

- 2.审计日志的查找
因为我们没有开启rbd客户端的日志，所以看不到qemu在调librbd的时候出现了什么问题，于是去Ceph服务端查看，我们在虚拟机挂掉的时间周围在ceph的审计日志中找到了这条记录
```
2020-03-12 23:53:11.646925 mon.0 127.0.0.1:6789/0 1565359 : audit [INF] from='client.? 127.0.0.1:0/3655019494' entity='client.cinder' cmd='[{"prefix": "osd blacklist", "addr": "127.0.0.1:0/1548601709", "blacklistop": "add"}]': finished
2020-03-13 00:11:20.690720 mon.0 127.0.0.1:6789/0 1566350 : audit [INF] from='client.? 127.0.0.1:0/2040474367' entity='client.cinder' cmd='[{"prefix": "osd blacklist", "addr": "127.0.0.1:0/1548601709", "blacklistop": "rm"}]': finished
2020-03-13 00:15:28.399958 mon.0 127.0.0.1:6789/0 1566589 : audit [INF] from='client.? 127.0.0.1:0/66127387' entity='client.cinder' cmd='[{"prefix": "osd blacklist", "addr": "127.0.0.1:0/2546996510", "blacklistop": "add"}]': finished
```
触发了把客户端加入黑名单的命令，也就是说qemu虚拟机的客户端被加入到了osd的黑名单，想想加入黑名单确实会导致qemu虚拟机在继续发送数据的时候会被拒绝，后面代码分析也是正确的，具体代码在下面解释

- 3.加入黑名单操作谁触发的
那到底谁触发了黑名单操作，就是找入口，看了一下大概有三个入口，分析如下：
- 入口1：librbd有一个break_lock接口，这个接口中会有可以能触发add blocklist
```
int break_lock(ImageCtx *ictx, const string& client,
		 const string& cookie)
{
  ...
  if (ictx->blacklist_on_break_lock) {
    ...
    r = rados.blacklist_add(client_address,
			      ictx->blacklist_expire_seconds);
    ...
  }
  ...
}
```
- 入口2：librados有add blocklist接口：
```
int blacklist_add(const std::string& client_address,
                      uint32_t expire_seconds);
```
- 入口3：在创建快照的请求一个回调函数中，如下：
```
template <typename I>
struct C_BlacklistClient : public Context {
  ...

  virtual void finish(int r) override {
    librados::Rados rados(image_ctx.md_ctx);
    r = rados.blacklist_add(locker_address,
                            image_ctx.blacklist_expire_seconds);
    on_finish->complete(r);
  }
};
```
前面两个入口只能是openstack或者qemu代码进行调用，查了一下代码没有，那就是第三个入口，而且我们确实是在创建快照的时候出现问题,而且我们从升级日志中也可以看出我们的请求是来自client.cinder用户，也即是来自外部调ceph，而不是ceph内部产生的，如果内部产生或者有谁命令行调用那用户应该是client.admin

- 4.创建快照的时候会什么会有这个回调函数
我们首先看一下这个回调类是在哪里创建的
```
template <typename I>
void BreakRequest<I>::send_blacklist() {
  ...
  m_image_ctx.op_work_queue->queue(new C_BlacklistClient<I>(m_image_ctx,
                                                            m_locker.address,
                                                            ctx), 0);
}
```
可以看到是在send_blacklist函数中创建的回调类，那这个函数是谁调用的呢？
```
template <typename I>
void BreakRequest<I>::handle_get_watchers(int r) {
  ...

  for (auto &watcher : m_watchers) {
    if ((strncmp(m_locker.address.c_str(),
                 watcher.addr, sizeof(watcher.addr)) == 0) &&
        (m_locker.handle == watcher.cookie)) {
      ldout(cct, 10) << "lock owner is still alive" << dendl;

      if (m_force_break_lock) {
        break;
      } else {
        finish(-EAGAIN);
        return;
      }
    }
  }

  send_blacklist();
}
```
在这个handle_get_watchers函数中掉了，这个函数主要是用来获取一个rbd上面的所有watcher，那watcher是什么呢？没跟rbd创建一个新的连接都会有一个watcher，也就是要获取这个rbd的所有客户端，我们可以通过分析创建快照的代码知道，创建快照要获取exclusive_lock锁，并且要知道这个锁现在被哪个watcher，也就是客户端给拥有这个，这样活把创建快照的请求分为local request和remote request交个锁的拥有者去执行创建快照元数据操作，而请求的所有者已经在创建连接refresh的时候已经保存到了m_locker中了，现在在获取watcher只是确认一下这个locker的owner是不是还存在。这里先等一下，我们想一下，在既有qemu客户端又有nova-compute客户端创建快照的操作的时候，因为虚拟机qemu是第一个操作这个rbd的，所以exclusive_lock的owner就是qemu进程。上面的代码的正常流程是nova-compute在创建rbd快照的时候，watcher会返回两个值，一个是qemu虚拟机客户端，一个nova-compute发起创建快照的这个自身客户端，而m_locker这个应该是qemu虚拟机这个客户端，在if判断的时候应该能匹配到qemu的客户端这个watcher，如过匹配到就是走下面的if和else，而m_force_break_lock是写死false
```
template <typename I>
void AcquireRequest<I>::send_break_lock() {
  ...
  auto req = BreakRequest<I>::create(
    m_image_ctx, m_locker, m_image_ctx.blacklist_on_break_lock, false, ctx);
  req->send();
}
```
就是上面创建的第四个参数，现在既然能走到send_blacklist，那就有两种可能，一种就是从osd那边返回的watcher是空的，没有进入for循环，一种就是没有匹配进入if也就是qemu虚拟机的watcher没了，猜想是第二种，那watcher为什么会没了呢？首先我们看一下watcher在osd那边是怎么存储的？

- 5. watcher持久化
watch持久化在前面的文章中我已经分析过是存储在rbd_header的文件对象的扩展属性中，如下：
```
[root@con02 0.ee_head]# attr -q -g "ceph._" rbd\\uheader.301027238e1f29__head_A5BD3CEE__0   > /root/object_info_t.txt
[root@con02 0.ee_head]# attr -q -g "ceph._@1" rbd\\uheader.301027238e1f29__head_A5BD3CEE__0   >> /root/object_info_t.txt
```
内容类似如下：
```
{
    "oid": {
        "oid": "rbd_header.301027238e1f29",
        "key": "",
        "snapid": -2,
        "hash": 2780642542,
        "max": 0,
        "pool": 0,
        "namespace": ""
    },
    ...

    "watchers": {
        "client.8079633": {
            "cookie": 139996606507840,
            "timeout_seconds": 30,
            "addr": {
                "nonce": 3613298630,
                "addr": "127.0.0.1:0"
            }
        }
    }
}
```
我们看到watchers是一个字典，当前只是示例，可以看到在watcher中有一个timeout_seconds字段，默认是30s，那这个字段是干嘛的呢，看名字就是超过30s删除，那是怎么删除

- 6.watcher的timeout
watcher就是客户端，其实客户端为了保活，会通过心跳ping来更新超时，如何做的呢，首先心跳在objecter的tick函数中每个5s调用
```
void Objecter::tick(){
  ...
  for (map<uint64_t,LingerOp*>::iterator p = s->linger_ops.begin();
	p != s->linger_ops.end();
	++p) {
      LingerOp *op = p->second;
      LingerOp::unique_lock wl(op->watch_lock);
      assert(op->session);
      ldout(cct, 10) << " pinging osd that serves lingering tid " << p->first
		     << " (osd." << op->session->osd << ")" << dendl;
      found = true;
      if (op->is_watch && op->registered && !op->last_error)
	_send_linger_ping(op);
    }
  ...    

}
```
通过_send_linger_ping就向osd发送心跳，以更新watcher的超时
```
void Objecter::_send_linger_ping(LingerOp *info){
  ...
	opv[0].op.op = CEPH_OSD_OP_WATCH;
  opv[0].op.watch.cookie = info->get_cookie();
  opv[0].op.watch.op = CEPH_OSD_WATCH_OP_PING;
  opv[0].op.watch.gen = info->register_gen;
	...

}
```
看到请求操作码是CEPH_OSD_OP_WATCH，这里还有一个watcher的操作码CEPH_OSD_WATCH_OP_PING，在osd端查看处理流程
```
int ReplicatedPG::do_osd_ops(OpContext *ctx, vector<OSDOp>& ops){
   ...
	 case CEPH_OSD_OP_WATCH:{
		 ...
		 else if (op.watch.op == CEPH_OSD_WATCH_OP_PING) {
	  ...
	  dout(10) << " found existing watch " << w << " by " << entity << dendl;
	  p->second->got_ping(ceph_clock_now(NULL));
	  result = 0;
        }
		 ...

	 }
	 ...
}
```
通过两个操作吗的匹配可以看到是got_ping进行处理，函数如下：
```
void Watch::got_ping(utime_t t)
{
  last_ping = t;
  if (conn) {
    register_cb();
  }
}
```
这里看到有注册回调
```
void Watch::register_cb()
{
  Mutex::Locker l(osd->watch_lock);
  if (cb) {
    dout(15) << "re-registering callback, timeout: " << timeout << dendl;
    cb->cancel();
    osd->watch_timer.cancel_event(cb);
  } else {
    dout(15) << "registering callback, timeout: " << timeout << dendl;
  }
  cb = new HandleWatchTimeout(self.lock());
  osd->watch_timer.add_event_after(
    timeout,
    cb);
}
```
上面就是watcher超时更新，大概意思就是如果有收到ping请求，就把原来的超时回调给删了，注册新的30s超时回调，通俗讲就是续命保活，好了到这里我们分析完了watcher超时更新，我们上面说过，加入加黑名单流程是因为没有匹配qemu的watcher，可以就是osd端没有收到ping心跳导致没有更细watcher的超时，然后watcher被删除了。而objecter的tick是每隔5s会调用一次，那为什么osd没有收到呢，那就回想一下我们开头的说的几个条件，osd磁盘性能很差，删除快照的时候产生大量的io，而tick发的心跳也是作为op请求网络发送到osd进行处理，那很有可能会出现osd在磁盘性能很差的情况下出现ping心跳请求被block，这也跟我们收到一分多钟的慢请求告警一致。好我们继续跟踪在触发了add blacklist之后，虚拟机的qemu进程是如何io block挂掉的

- 7.add blacklist流程
创建快照时的回调类在触发add blacklist时，请求是发现mon的，mon在收到请求之后通过paxos，修改osdmap，把黑名单记录在了osdmap里面，由于osdmap的改变，osdmap的版本会加1，从而扩散到各个osd,下面是版本加1的osdmap
```
{
	...
	"pg_temp": [],
  "blacklist": {
		"127.0.0.1:0/147234759": "2020-03-15 22:20:44.052504"
  },
  ...
}
```

- 7.qemu读写请求继续操作
qemu虚拟机根本不知道自己被加入了黑名单，还是会调用librbd继续发送读写请求，那我们看看osd那边是怎么处理
```
void ReplicatedPG::do_op(OpRequestRef& op)
{
	...
	// blacklisted?
  if (get_osdmap()->is_blacklisted(m->get_source_addr())) {
    dout(10) << "do_op " << m->get_source_addr() << " is blacklisted" << dendl;
    osd->reply_op_error(op, -EBLACKLISTED);
    return;
  }
	...
}
```
在osd的do_op函数中检查请求来的客户端是不是在黑名单中，是则返回EBLACKLISTED错误码，而这个错误码是108，也即是shutdown错误，所以我们看到qemu虚拟机日志中出现的108错误。
至此我们我们就分析完了

# 解决方法
那有什么可以避免在watcher失效之后，不执行add blacklist，可以修改一个参数
```
template <typename I>
void BreakRequest<I>::send_blacklist() {
  if (!m_blacklist_locker) {
    send_break_lock();
    return;
  }

  CephContext *cct = m_image_ctx.cct;
  ldout(cct, 10) << dendl;

  // TODO: need async version of RadosClient::blacklist_add
  using klass = BreakRequest<I>;
  Context *ctx = create_context_callback<klass, &klass::handle_blacklist>(
    this);
  m_image_ctx.op_work_queue->queue(new C_BlacklistClient<I>(m_image_ctx,
                                                            m_locker.address,
                                                            ctx), 0);
}
```
可以看到在生成加黑名单的回调类之前，如果m_blacklist_locker为false，则可以跳过，这个参数是通过配置rbd_blacklist_on_break_lock赋予的，这个配置默认是True，所以修改配置为false，则可以避免add blacklist操作
