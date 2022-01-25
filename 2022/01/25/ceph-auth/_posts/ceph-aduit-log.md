---
title: ceph审计日志知多少
date: 2020-03-15 15:23:50
tags: ceph log
---

# 问题
我们知道一般系统都会有审计日志，这个审计日志是用来记录系统的一些操作，这样可以方便后面排查问题以及观察集群状态。ceph作为一个分布式存储系统毫无例外，也有这个审计日志，我们先来看看ceph的审计日志内容：
```
2020-03-12 23:53:11.646925 mon.0 127.0.0.1:6789/0 1565359 : audit [INF] from='client.? 127.0.0.1:0/3655019494' entity='client.admin' cmd='[{"prefix": "osd blacklist", "addr": "127.0.0.1:0/1548601709", "blacklistop": "add"}]': finished
2020-03-13 00:11:20.690720 mon.0 127.0.0.1:6789/0 1566350 : audit [INF] from='client.? 127.0.0.1:0/2040474367' entity='client.admin' cmd='[{"prefix": "osd blacklist", "addr": "127.0.0.1:0/1548601709", "blacklistop": "rm"}]': finished
2020-03-13 00:15:28.399958 mon.0 127.0.0.1:6789/0 1566589 : audit [INF] from='client.? 127.0.0.1:0/66127387' entity='client.admin' cmd='[{"prefix": "osd blacklist", "addr": "127.0.0.1:0/2546996510", "blacklistop": "add"}]': finished
```
可以看到这是一条把某一客户端加黑名单的命令，下面就来分析一下，这条命令是怎么纪律到ceph的审计日志中的

# 分析
上面的命令我们是使用ceph osd blacklist add命令执行生成的，ceph是一个python脚本，这里不做过多的分析，直接告诉你这个命令是发送到了mon，下面我们来分析一下mon那端的处理
- 1.mon接受请求
```
else if (prefix == "osd blacklist") {
    string addrstr;
    cmd_getval(g_ceph_context, cmdmap, "addr", addrstr);
    entity_addr_t addr;
    if (!addr.parse(addrstr.c_str(), 0)) {
      ss << "unable to parse address " << addrstr;
      err = -EINVAL;
      goto reply;
    }
    else {
      string blacklistop;
      cmd_getval(g_ceph_context, cmdmap, "blacklistop", blacklistop);
      if (blacklistop == "add") {
	utime_t expires = ceph_clock_now();
	double d;
	// default one hour
	cmd_getval(g_ceph_context, cmdmap, "expire", d,
          g_conf->mon_osd_blacklist_default_expire);
	expires += d;

	pending_inc.new_blacklist[addr] = expires;

        {
          // cancel any pending un-blacklisting request too
          auto it = std::find(pending_inc.old_blacklist.begin(),
            pending_inc.old_blacklist.end(), addr);
          if (it != pending_inc.old_blacklist.end()) {
            pending_inc.old_blacklist.erase(it);
          }
        }

	ss << "blacklisting " << addr << " until " << expires << " (" << d << " sec)";
	getline(ss, rs);
	wait_for_finished_proposal(op, new Monitor::C_Command(mon, op, 0, rs,
						  get_last_committed() + 1));
	return true;
      } else if (blacklistop == "rm") {
	if (osdmap.is_blacklisted(addr) ||
	    pending_inc.new_blacklist.count(addr)) {
	  if (osdmap.is_blacklisted(addr))
	    pending_inc.old_blacklist.push_back(addr);
	  else
	    pending_inc.new_blacklist.erase(addr);
	  ss << "un-blacklisting " << addr;
	  getline(ss, rs);
	  wait_for_finished_proposal(op, new Monitor::C_Command(mon, op, 0, rs,
						    get_last_committed() + 1));
	  return true;
	}
	ss << addr << " isn't blacklisted";
	err = 0;
	goto reply;
      }
    }
  }
```
在osdmonitor具体处理请求这块，在osdmap中增加黑名单这块没有啥审计日志记录，如果是我我也会觉得审计日志应该在所有请求的总入口，然后对请求进行审计记录。在monitor类的handle_command中找到了大致代码
```
void Monitor::handle_command(MonOpRequestRef op)
{

 ...
(cmd_is_rw ? audit_clog->info() : audit_clog->debug())
    << "from='" << session->inst << "' "
    << "entity='" << session->entity_name << "' "
    << "cmd=" << m->cmd << ": dispatch";

  ...
}
```
上面的audit_clog就是升级日志模块客户端。

- 2.audit_clog初始化
首先我们看一下audit_clog是什么时候初始化的
```
mon = new Monitor(g_ceph_context, g_conf->name.get_id(), store,
		    msgr, mgr_msgr, &monmap);
```
在ceph-mon的main函数中，会生成monitor类对象，在这个类的构造函数中会初始化audit_clog
```
{
  clog = log_client.create_channel(CLOG_CHANNEL_CLUSTER);
  audit_clog = log_client.create_channel(CLOG_CHANNEL_AUDIT);

  update_log_clients();

  paxos = new Paxos(this, "paxos");
```
可以看出这里除了创建audit_clog外还创建了clog，可以看出他们是一样的只是名字不一样，clog是集群日志。通过log_client我们可以猜出来这个这mon是定义了一个日志的客户端，那既然有客户端，也肯定有服务端，那服务端是什么呢？先说一下是monitor中的logmonitor，后面会详细讲解，这里看出来对于系统日志和审计日志使用了C/S模式，主要了为了一个目的日志的统一收集，以及一致性。
西面我们继续看audit_clog的初始化，看log_client是创建了一个通道channel，如下
```
LogChannelRef create_channel(const std::string& name) {
    LogChannelRef c;
    if (channels.count(name))
      c = channels[name];
    else {
      c = std::make_shared<LogChannel>(cct, this, name);
      channels[name] = c;
    }
    return c;
  }
```
创建了一个LogChannel对象，并缓存在log_client的channels这个map中。LogChannel对象主要是定了日志各种日志等级的操作，如下所示：
```
LogClientTemp info() {
    return LogClientTemp(CLOG_INFO, *this);
  }
  void info(std::stringstream &s) {
    do_log(CLOG_INFO, s);
  }
```
这个是输出info基本的日志，也就是我们在审计日志中看到的
```
[INF]
```
上面便是初始化，我们也发现审计日志的客户端只在mon有，osd和mds都没有log_client，所以审计日志收集的日志mon那边的请求命令记录

- 3.审计日志记录流程
- （1）.日志级别
前面讲完了升级日志客户端的初始化，现在下面跟一下审计日志记录流程，首先看命令入口处的日志记录
```
(cmd_is_rw ? audit_clog->info() : audit_clog->debug())
    << "from='" << session->inst << "' "
    << "entity='" << session->entity_name << "' "
    << "cmd=" << m->cmd << ": dispatch";
```
这里便是一条升级日志的记录，从上面可以看出，对于审计日志的info和debug的日志级别是根据cmd_is_rw这个是否为true进行选择的，从字面上看是这个命令是读写的吗,难道命令含有读写属性？不错是的，我们具体看一个命令
```
COMMAND("osd blacklist " \
	"name=blacklistop,type=CephChoices,strings=add|rm " \
	"name=addr,type=CephEntityAddr " \
	"name=expire,type=CephFloat,range=0.0,req=false", \
	"add (optionally until <expire> seconds from now) or remove <addr> from blacklist", \
	"osd", "rw", "cli,rest")
```
这是mon那边定义的osd blacklist的命令，可以看到第四个参数"rw"，这边是命令的读写属性，搞明白了日志级别之后，我们这里是rw，所以使用audit_clog->info（）函数来输出后面的字符串日志记录，看一下info函数

- (2)日志打印
```
LogClientTemp info() {
    return LogClientTemp(CLOG_INFO, *this);
  }
```
info函数返回LogClientTemp类对象，看看构造函数
```
LogClientTemp::LogClientTemp(clog_type type_, LogChannel &parent_)
  : type(type_), parent(parent_)
{
}
```
没做啥，只是记录了日志级别，难道就这样完了？日志怎么写入的呢，这里使用了一个技巧，利用了作用域加析构函数，我们看到info函数结束return会触发LogClientTemp调用自己的析构函数，我们看看他的析构函数做了啥
```
LogClientTemp::~LogClientTemp()
{
  if (ss.peek() != EOF)
    parent.do_log(type, ss);
}
```
调用了parent的do_log,这个parent就是audit这个channel，看一下他的do_log函数
```
void LogChannel::do_log(clog_type prio, std::stringstream& ss)
{
  while (!ss.eof()) {
    string s;
    getline(ss, s);
    if (!s.empty())
      do_log(prio, s);
  }
}

void LogChannel::do_log(clog_type prio, const std::string& s)
{
  Mutex::Locker l(channel_lock);
  int lvl = (prio == CLOG_ERROR ? -1 : 0);
  ldout(cct,lvl) << "log " << prio << " : " << s << dendl;
  ...

  // log to monitor?
  if (log_to_monitors) {
    e.seq = parent->queue(e);
  } else {
    e.seq = parent->get_next_seq();
  }

  ...
}
```
第一个do_log函数调用了第二个do_log重载函数，为什么是第一个调用二个，因为LogChannel重载了<<运算符，如下：
```
std::ostream& operator<<(const T& rhs)
  {
    return ss << rhs;
  }

stringstream ss;
```

- （3）do_log
继续看一下do_log
```
void LogChannel::do_log(clog_type prio, const std::string& s)
{
  Mutex::Locker l(channel_lock);
  int lvl = (prio == CLOG_ERROR ? -1 : 0);
  ldout(cct,lvl) << "log " << prio << " : " << s << dendl;
  ...

  // log to monitor?
  if (log_to_monitors) {
    e.seq = parent->queue(e);
  } else {
    e.seq = parent->get_next_seq();
  }

  ...
}
```
这里有两个关注点第一个mon本地的日志的打印
```
ldout(cct,lvl) << "log " << prio << " : " << s << dendl;
```
这个日志是记录到mon的本地日志中,也就是我们的mon.{host-name}.log
第二个是
```
// log to monitor?
if (log_to_monitors) {
  e.seq = parent->queue(e);
} else {
  e.seq = parent->get_next_seq();
}
```
默认log_to_monitors是true，所以会发送到mon也就是我们上面的说的服务端logmonitor,看一下审计日志是如何发送的

- （4）审计日志发送到mon
```
version_t LogClient::queue(LogEntry &entry)
{
  Mutex::Locker l(log_lock);
  entry.seq = ++last_log;
  log_queue.push_back(entry);

  if (is_mon) {
    _send_to_mon();
  }

  return entry.seq;
}
```
首先进去queue，然后判断是不是is_mon，因为审计日志是在mon上初始化的所以这个是true，然后使用_send_to_mon发送给mon
```
void LogClient::_send_to_mon()
{
  assert(log_lock.is_locked());
  assert(is_mon);
  assert(messenger->get_myname().is_mon());
  ldout(cct,10) << __func__ << "log to self" << dendl;
  Message *log = _get_mon_log_message();
  messenger->get_loopback_connection()->send_message(log);
}
```
看到生成了一个日志消息，这个消息的类型是MSG_LOG，我们直接略过消息的发送和接受，直接通过消息类型定位mon的处理函数

- （5）logmonitor处理
logmonitor处理经历LogMonitor::preprocess_query和LogMonitor::prepare_update，前面一个没做啥处理，看一下prepare_update
```
bool LogMonitor::prepare_update(MonOpRequestRef op)
{
  op->mark_logmon_event("prepare_update");
  PaxosServiceMessage *m = static_cast<PaxosServiceMessage*>(op->get_req());
  dout(10) << "prepare_update " << *m << " from " << m->get_orig_source_inst() << dendl;
  switch (m->get_type()) {
  case MSG_MON_COMMAND:
    return prepare_command(op);
  case MSG_LOG:
    return prepare_log(op);
  default:
    ceph_abort();
    return false;
  }
}
```
通过消息类型又调用prepare_log
```
bool LogMonitor::prepare_log(MonOpRequestRef op)
{

  ...

  for (deque<LogEntry>::iterator p = m->entries.begin();
       p != m->entries.end();
       ++p) {
    dout(10) << " logging " << *p << dendl;
    if (!pending_summary.contains(p->key())) {
      pending_summary.add(*p);
      pending_log.insert(pair<utime_t,LogEntry>(p->stamp, *p));
    }
  }
  pending_summary.prune(g_conf->mon_log_max_summary);
  wait_for_finished_proposal(op, new C_Log(this, op));
  return true;
}
```
这里加入到了pending_summary，和pending_log，logmonitor继承了paxos，经过决策来最终确定审计日志数据的一致性，这类不过多解释paxos，我们直接看写入部分


- （6）审计日志写入审计日志文件
根据paxos框架最终会调用update_from_paxos
```
void LogMonitor::update_from_paxos(bool *need_bootstrap)
{
  ...

  map<string,bufferlist> channel_blog;

  ...

  for(map<string,bufferlist>::iterator p = channel_blog.begin();
      p != channel_blog.end(); ++p) {

  ...

  int fd = ::open(log_file.c_str(), O_WRONLY|O_APPEND|O_CREAT|O_CLOEXEC, 0600);
  ...
  int err = p->second.write_fd(fd);

  ...
  }
}    
```
channel_blog收集要写入的日志，open打开审计日志文件名，然后write_fd写入。
对于审计日志的文件名默认是ceph.audit.log，对于集群日志clog也是类似的分析
