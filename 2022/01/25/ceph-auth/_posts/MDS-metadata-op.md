---
title: MDS元数据请求操作流程
date: 2021-03-11 21:35:04
tags: Ceph
---

# 小记
这篇是学生时代写的笔记

# 概要
本文章主要是讲解两种元数据请求：
（1）需要修改元数据的请求；
（2）不需要修改的元数据请求。解析元数据请求的接收，处理以及回复客户端

# MDS元数据请求处理流程
- 1.接受元数据请求
 首先元数据的请求是client发出来的，充当client角色的那台电脑通过simplemessage模块发送到网络上，然后充当MDS角色的那台电脑通过simplemssage从网络中接收消息，然后分发给那台电脑的MDS模块。这些是网络模块消息的传递，你应该懂吧！不懂我要打屁屁了。
 接着上面，在MDS模块接收到消息之后我们看一下代码的流程：
 ```
 MDS::ms_dispatch（）--> MDS::_dispatch(Message *m)
 ```
 这个函数式接收消息，对于接收到的消息可能有很多，因为client，Mon，OSD和其他的MDS都可以和这台MDS都可以进行消息的通信，对于这些消息有分为两大类：（1）优先级比较高的，主要是一些map信息这些map记录了集群的很多信息，比如MDS集群，OSD集群和MON集群；（2）优先级比较低的消息，client，Mon，OSD和其他的MDS发送过来的消息这些就是属于这种。代码如下（在MDS::_dispatch（）函数中）：
 ```
 bool MDS::_dispatch(Message *m)
{
// core处理消息的重点
  if (!handle_core_message(m))//优先级比较的高的消息
  {
    if (is_laggy())
	{
      dout(10) << " laggy, deferring " << *m << dendl;
      waiting_for_nolaggy.push_back(m);
    }
	else
	{
      if (!handle_deferrable_message(m))  // 优先级低的消息
	  {
	    dout(0) << "unrecognized message " << *m << dendl;
	    m->put();
	    return false;
      }
    }
  }
}
 ```
 上面已标出两大类消息类型,我们这里主要是讨论client发过来的元数据请求属于优先级低的消息,所以代码流程如下：
 ```
 MDS::_dispatch(Message *m) --> MDS::handle_deferrable_message(Message *m)
 ```
 在handle_deferrable_message()函数中根据消息的类型标识，知道需要Server这个模块进行代码的出来，如下:
 ```
 bool MDS::handle_deferrable_message(Message *m)
{
 ...
case CEPH_MSG_CLIENT_SESSION:
case CEPH_MSG_CLIENT_RECONNECT:
case CEPH_MSG_CLIENT_REQUEST:  //消息标识显示出这事处理客户端的元数据请求
      server->dispatch(m);
...
}
 ```
 从上面可以看出对于client端传过来的请求：主要分为三类（就是三个case）：
 （1）client的会话
 （2）client的重连接
 （3）客户端的元数据请求，这三种都是客户端的请求，都要Server模块进行处理，具体看如下：
 ```
 void Server::dispatch(Message *m)
{
   ....
   switch (m->get_type()) {
  case CEPH_MSG_CLIENT_SESSION:
    handle_client_session(static_cast<MClientSession*>(m));
    return;
  case CEPH_MSG_CLIENT_REQUEST:    //这主要是处理client的元数据请求
    handle_client_request(static_cast<MClientRequest*>(m));
    return;
  case MSG_MDS_SLAVE_REQUEST:
    handle_slave_request(static_cast<MMDSSlaveRequest*>(m));
    return;
  }
}
 ```
 由于上面Server主要处理三种client端发过来的消息，而对于客户端的元数据请求主要是handle_client_request()函数。代码流程如下：
 ```
 server->dispatch(m) --> Server::handle_client_request(MClientRequest *req)
 ```
 handle_client_request（）主要是做对元数据请求消息的注册和缓存，就是在MDS中缓存处理过的元数据请求消息，然后对client的元数据请求消息的具体处理交给Server::dispatch_client_request(MDRequestRef& mdr)函数处理；代码流程如下：
 ```
 Server::handle_client_request(MClientRequest *req) --> Server::dispatch_client_request(MDRequestRef& mdr)
 ```
 dispatch_client_request函数可以从函数的名字可以看出是分发client端的消息，因为client的元数据请求操作有好多，比如打开文件，删除文件，修改文件属性等等，所以这个函数就是用来分发具体的消息。

 我们看一下大致有哪些元数据请求？ 一共有27中元数据请求,分别如下
```
CEPH_MDS_OP_LOOKUPHASH
CEPH_MDS_OP_LOOKUPINO
CEPH_MDS_OP_LOOKUPPARENT
CEPH_MDS_OP_LOOKUPNAME
CEPH_MDS_OP_LOOKUP
CEPH_MDS_OP_LOOKUPSNAP
CEPH_MDS_OP_GETATTR
CEPH_MDS_OP_SETATTR
CEPH_MDS_OP_SETLAYOUT
CEPH_MDS_OP_SETDIRLAYOUT
CEPH_MDS_OP_SETXATTR
CEPH_MDS_OP_RMXATTR
CEPH_MDS_OP_READDIR
CEPH_MDS_OP_SETFILELOCK
CEPH_MDS_OP_GETFILELOCK
CEPH_MDS_OP_CREATE
CEPH_MDS_OP_OPEN
CEPH_MDS_OP_MKNOD
CEPH_MDS_OP_LINK
CEPH_MDS_OP_UNLINK
CEPH_MDS_OP_RMDIR
CEPH_MDS_OP_RENAME
CEPH_MDS_OP_MKDIR
CEPH_MDS_OP_SYMLINK
CEPH_MDS_OP_LSSNAP
CEPH_MDS_OP_MKSNAP
CEPH_MDS_OP_RMSNAP
```
 这就是27个元数据请求，具体哪个请求是对目录操作，哪个是对文件操作，可以通过名字看出，这里也使用了一种状态机的原理，每一种请求就是一种状态，但是这些状态时有限的，状态之间是不能转化的。

- 2.元数据请求处理
分为需不需要修改元数据主要是需要修改元数据的请求会生成日志，没有修改的元数据请求不需生成日志

- 2.1不需修改元数据的请求处理
这里我们以open元数据操作为例就是打开一个文件，这个操作是不需要进行元数据的修改的，相应的元数据请求类型为CEPH_MDS_OP_OPEN，处理函数为handle_client_open()。下面看一下其关键代码：
```
void Server::handle_client_open(MDRequestRef& mdr)
{
MClientRequest *req = mdr->client_request;//从函数操作中取出client的元数据请求
...
CInode *cur = rdlock_path_pin_ref(mdr, 0, rdlocks, need_auth);//这个是从MDCache中查找相关元数据
...
reply_request(mdr, 0, cur, dn);//返回回复给client
}
```
可以看出这里最重要的是从MDCache中查找相关的元数据。我们来看一下rdlock_path_pin_ref()函数
```
Server::rdlock_path_pin_ref() --> MDCache::path_traverse()
```
Path_traverse()函数才是最终查找元数据函数。对于元数据的查找我之前跟你说过了，不记得我论文呢上也有图。就是按Cdir->CDentry->Cinode这个顺序查找元数据的。
由于open操作没有修改元数据所以没有产生日志，直接可以给client回复一个消息就可以，这就是reply_request()函数。这个函数会先生成一个回复消息，然后通过simplemessage发送到client上。

- 2.2需修改元数据的请求处理
上面没有修改元数据请求的操作时比较简单的，而需要修改元数据的操作就比较复杂，因为会生成相应的日志事件，这样就增加了日志的管理。这里我们以CEPH_MDS_OP_SETATTR这种元数据请求类型的操作为例，它的处理函数为handle_client_setattr()，这个元数据请求主要是修改文件的属性，比如修改文件的大小，所属者等等，具体的相关代码如下：
```
void Server::handle_client_setattr(MDRequestRef& mdr)
{
MClientRequest *req = mdr->client_request;//首先从函数操作中取出元数据请求
...
CInode *cur = rdlock_path_pin_ref(mdr, 0, rdlocks, true);//在MDCache中查找元数据
...
mdr->ls = mdlog->get_current_segment();
EUpdate *le = new EUpdate(mdlog, "setattr");//生成相关日志
mdlog->start_entry(le);
...
pi = cur->project_inode(); //写时拷贝生成一个元数据的副本
...
if (mask & CEPH_SETATTR_MODE)  //修改元数据副本的属性
    pi->mode = (pi->mode & ~07777) | (req->head.args.setattr.mode & 07777);
 if (mask & CEPH_SETATTR_UID)
    pi->uid = req->head.args.setattr.uid;
if (mask & CEPH_SETATTR_GID)
    pi->gid = req->head.args.setattr.gid;
if (mask & CEPH_SETATTR_MTIME)
    pi->mtime = req->head.args.setattr.mtime;
if (mask & CEPH_SETATTR_ATIME)
    pi->atime = req->head.args.setattr.atime;
if (mask & (CEPH_SETATTR_ATIME | CEPH_SETATTR_MTIME))
    pi->time_warp_seq++;  
...//日志的处理并回复client
journal_and_reply(,,new C_MDS_inode_update_finish(mds, mdr, cur,
							truncating_smaller, changed_ranges))
}
```
首先可以看出前面处理的部分跟不需修改元数据的请求时一样的，都要从MDCache中国把元数据查找出来。不同的就是后面生成相关的日志以及如何修改元数据。
这里对元数据的修改使用的是COW（写时拷贝技术），就是复制一个副本，然后再副本上面修改元数据，修改后的副本不会立即替换掉原来的元数据，也不会把原来的元数据标记为脏，这些修改的元数据副本会放在一个队列中。这个我们从副本元数据怎样生成的久可以看出，就是CInode::project_inode()函数。
```
inode_t *CInode::project_inode(map<string,bufferptr> *px)
{//在projected_nodes最后面增加一个projected_nodes存的是指向同一个INOde 的多个脏版本
  if (projected_nodes.empty()) {  //空的时候增加一个本inode 的
    projected_nodes.push_back(new projected_inode_t(new inode_t(inode)));
    if (px)
      *px = xattrs;
  } else {  //不为空时把最后一个复制再插入到最后
    projected_nodes.push_back(new projected_inode_t(
        new inode_t(*projected_nodes.back()->inode))); //inode号是一样的
    if (px)
      *px = *get_projected_xattrs();  //赋值一个最新的脏数据版本信息
  }
  projected_nodes.back()->xattrs = px;
  dout(15) << "project_inode " << projected_nodes.back()->inode << dendl;
  return projected_nodes.back()->inode;
}
```
这里我们首先要介绍一下副本元数据的数据结构类型projected_inode_t，代码如下：
```
struct projected_inode_t {//副本元数据数据版本信息
    inode_t *inode;
    map<string,bufferptr> *xattrs;//这里存的就是修改的数据
    sr_t *snapnode;

    projected_inode_t()
      : inode(NULL), xattrs(NULL), snapnode(NULL) {}
    projected_inode_t(inode_t *in, sr_t *sn)
      : inode(in), xattrs(NULL), snapnode(sn) {}
    projected_inode_t(inode_t *in, map<string, bufferptr> *xp = NULL, sr_t *sn = NULL)
      : inode(in), xattrs(xp), snapnode(sn) {}
  };
```
然后在Cinode类中一个projected_inode_t的链表，代码如下：
```
Class Cinode
{
   list<projected_inode_t*> projected_nodes;//所有的副本元数据链表
}；
```
Cinode中使用链表来管理所有的副本元数据，并且这是有顺序的，链表从头到尾所管理的副本元数据越新，什么意识？就是越新生产的副本元数据越在链表尾。

在修改了副本元数据之后，就会生成相关的日志事件，这里生成的日志事件为Eupdate，从名字可以看出这个是一个更新的日志事件，日志生成完之后就会把日志交给所管理的LogSegment，就是说这个日志事件交给哪个LogSegment管理，这个主要是通过下面这个函数实现：
```
void MDLog::start_entry(LogEvent *e)//这个日志时间要要写在哪个地方
{
  assert(cur_event == NULL);
  cur_event = e;//日志模块中一个指向当前日志事件 的指针
  e->set_start_off(get_write_pos());  //从一个日志管理器journaler中得到下一个日志事件开始写的位置，并设置给当前的日志事件
}
```
为什么要LogSegment？首先说明一下，日志最终肯定是要写到OSD上的日志文件中，日志要顺序写，这些都是写到底层物理磁盘上。而我们这里的LogSegment是MDS中的，这个是逻辑上面管理日志文件，可以想象成日志文件由一个接一个LogSegment连接而成，每一个LogSegment长度为4M，而每一个LogSegment是有一个个日志事件组成。日志都是一个个在LogSegment中往后追加，就是后生成的日志在后面，等一个LogSegment满了一个4M大小之后，就生成一个新的空的LogSegment存放后面的日志。所以也可以看出日志是以LogSegment为单位写到OSD上去的。

下面我们要介绍一下一个重要的关于日志写到OSD上之后，如何异步继续处理下一步，这里使用的是回调函数，但是ceph中使用类包装了回调函数，生成了一个回调类。至于回调类看以看具体的《Context回调类》文档。上面这个handle_client_setattr元数据请求函数就生成了一个回调类C_MDS_inode_update_finish，从这个类的名字可以看出实在日志写到OSD上之后需要进行update操作，具体就是使用修改后的副本元数据替换掉原来的元数据，并把元数据标记为脏。代码如下
```
journal_and_reply(,,new C_MDS_inode_update_finish(mds, mdr, cur,
							truncating_smaller, changed_ranges))
```
蓝色字就是生成一个回调类对象，具体的操作就是进入这个journal_and_reply函数执行，从名字可以看出是要执行日志的处理和回复client两个部分。

下面看一下journal_and_reply具体代码：
```
void Server::journal_and_reply
{
...
early_reply(mdr, in, dn);  //对client端进行早期回复
...
mdlog->submit_entry(le, fin);//这个主要是把刚刚生成的日志写到日志写缓存中
...
}
```
这里要解释两个：
（1）	对client的早期回复：对client的早期回复，是尽量的回复给client，让client继续做后面的事，但是这时日志还没有写到OSD上，所以是不安全的，如果突然某一台MDS挂了，就不能进行回复。但是MDS挂了的概率还是比较少的，所以使用早期回复client，会大大较少元数据操作响应的延时。
（2）	把日志写到日志的写缓存中：日志生成之后，需要写到MDLog模块的一个日志写缓存中，你可以把这个缓存看成字符数组，就是把一个个日志对象序列化到这个字符数组中，至于什么是序列化操作？这个你自己上网百度一下。

下面具体看一下如何把日志写到日志缓存中的，首先看一下代码流程：
```
MDLog::submit_entry()  Journaler::append_entry() write_buf.claim_append(bl)
```
其中write_buf就是日志的写缓存，你可以看成是一个字符数组。
下面具体看一下Journaler::append_entry函数。
```
uint64_t Journaler::append_entry(bufferlist& bl)
{
...
// append把日志写到日志写缓存中
::encode(s, write_buf);//加入日志的长度值
write_buf.claim_append(bl);//加入日志的内容
write_pos += sizeof(s) + s;

...
uint64_t su = get_layout_period();//OSD对象大小，一般为4M
uint64_t write_off = write_pos % su; //计算日志是不是满一个4M大小
uint64_t write_obj = write_pos / su;
uint64_t flush_obj = flush_pos / su;
...
if (write_obj != flush_obj) //如果日志满足一个4M大小就要把日志写到OSD上
{
    ldout(cct, 10) << " flushing completed object(s) (su " << su << " wro " << write_obj << " flo " << flush_obj << ")" << dendl;
    _do_flush(write_buf.length() - write_off);
}

}
```
这里有一个write_pos，从名字可以看出而是一个显示位置的标志，这个位置是一直往后增的，就像打开文件之后有一个文件的pos位置一样。
这里主要介绍一下如何判断日志有没有满一个4M对象大小，这里首先识别出OSD上对象的大小，这里使用的是get_layout_period函数。计算是不是满足一个4M大小，这里使用了两个位置标识，write_obj和flush_obj，其中write_obj标识的是现在write_pos属于哪一个4M对象，flush_pos表示的是前面已经写到OSD上的日志位置，flush_obj表示的是flush_pos在哪一个4M对象，如果这两个值不相等，就表示在flush_pos和write_pos之间存在一个满足4M大小的对象没有写到OSD上，那此时如果满了一个4M大小的对象，就需要写到OSD上。具体的写到OSD上调用_do_flush()。
```
Journaler::_do_flush() -> Filer::write()
```
我们的流程先暂时到这里，其中Filer是OSDC模块中的一个文件管理模块。那OSDC是什么模块呢？OSDC就是OSD client的缩写，只要跟OSD交互的单元，比如client，MDS都需要这个OSDC模块，这个模块是CRUSH计算定位数据在OSD上的核心，比如client需要把文件的数据写到OSD上，这个是需要CRUSH算法分布文件的数据；MDS中的元数据和日志也都是存储在OSD上，所以也需要CRUSH算法分布这些数据。

- 2.3回调类注册
下面我们讲一下回调类的注册，之前我们在《Context回调类》文档中已经介绍了回调函数，这里我们介绍一下我们的回调函数是怎么注册和启动的。我们从MDLog::submit_entry函数看起，代码如下：
```
void MDLog::submit_entry(LogEvent *le, Context *c)
{
   ...
   if (c)  //如果有回调类就进行注册
    journaler->wait_for_flush(c);
   ...
}
```
这个回调类对象就是在处理元数据请求时生成的，这里通过journaler模块把日志进行注册，那问题来了，日志是怎么进行注册的？我们先看一下wait_for_flush(c)函数。
```
void Journaler::wait_for_flush(Context *onsafe)
{
   ...
   // queue waiter
  if (onsafe) //关键是这个
    waitfor_safe[write_pos].push_back(onsafe);
  ...
}
```
首先介绍一下waitfor_safe是一个map，代码如下：
```
std::map<uint64_t, std::list<Context*> > waitfor_safe;
```
建立一个位置对应一个链表的回调类，什么意识呢？就是说回调函数的注册注册在日志位置上，
              Flush_pos                write_pos
------------------|------------------------|-------------------------------------------------------------
前                                     List<context>                                    后
上面这个图是日志的逻辑位置，我们可以看出回调类是挂在日志所在逻辑位置的一个list中。

- 2.4回调函数的启动
回调类注册了，那什么时候回调类启动呢？在我们把日志写到OSD上之后，OSD会发生一个回复消息给MDS，MDS会更新flush_pos位置标志，这个标志表示在这个位置之前的日志都已经写到OSD上了，在flush_pos位置往后更新的时候，在flush_pos位置之前的所有注册的回调类对象就可以启动了。启动回调类对象就是调用对象的complete函数，然后调用finish这个回调函数，具体代码为忘记在哪里了。

- 3.回调函数的后续操作
回调函数的后续操作主要做什么？主要是把修改的副本元数据替换掉原来的元数据，并标记为脏。我们来看一下handle_client_setattr操作生成的回调类C_MDS_inode_update_finish，直接看一下这个回调类对象的finish函数。
```
void finish(int r)
{
   ...
   in->pop_and_dirty_projected_inode(mdr->ls);//把修改的副本元数据替换原来的元数据
   mdr->apply();//标记替换后的元数据为脏
   ...
   mds->server->reply_request(mdr, 0);//第二次向client回复消息
   ...
}
```
这里主要是介绍一下mdr->apply函数的调用流程:
```
mdr->apply() --> MutationImpl::apply()
```
这个标记脏的过程是，Cinode为脏，然后CDentry为脏，然后所在的父文件夹为脏。

- 3.1第二次回复client端
之前在处理日志之前，先回复了一次给client，不过那次回复是不安全的，日志还没有写到OSD上，这次的回复是安全的，日志已经写到OSD上。

- 4.定期清理日志
日志在写到OSD上之后，在MDS中的日志就会标记为过期的，这些过期的日志就可以删除了，以节省MDS中的内存。下面看一下MDS的定期清理函数：
```
void MDS::tick()
{
   ...
   mdcache->trim();//定期清理缓存中的元数据
   ...
   mdlog->trim();//定期清理过期的日志
}
```
(1)MDCache的清理主要是删除一些优先级比较低的元数据，腾出一些内存空间，存放新的元数据，我们知道元数据的CDentry是使用LRU管理的，这个MDCache的清理主要就是对这个LRU进行清理。这个是MDS系统定时进行主动的清理。其实还有一个被动的清理，比如我们从OSD上取出元数据要缓存在MDS中，发现LRU的空间不够了，这是就需要删除掉LRU中优先级比较低的元数据，这个就是被动清理。至于具体MDCache是怎么清理的，我们后面在元数据管理方面的文章中给予解释
(2)定期清理过期的日志：主要是在日志写到OSD上之后，MDS上面的那些日志就没有用了，我们需要删除以释放内存空间

下面我们来看一下具体怎么删除过期日志的。还记得我之前说过的日志是以LogSegment为逻辑单位进行管理的吗。这里还是以LogSegment为管理单元，对于已经写到OSD的日志，我们就标记这个日志对应的LogSegment为过期的，看代码如下：
```
void MDLog::trim(int m)
{
   ...
   while (p != segments.end())//遍历LogSegment集合
   {
      ...
      try_expire(ls, op_prio);//把过期的LogSegment放到过期LogSegment集合中
      ...
   }
_trim_expired_segments();//删除过期LogSegment
}
```
这里主要是看上面两个函数:
(1)try_expire()是把LogSegment变成过期可以删除的，在这个过程中需要把这个LogSegment记录的脏的元数据更新到OSD上。
(2)_trim_expired_segments函数这个函数删除过期LogSegment的具体操作，就是把过期的LogSegment从LogSegment集合中删除。

- 4.1更新元数据
更新脏元数据，每一个LogSegment中记录了脏的元数据，可以再回顾一下LogSegment的类内容：
```
class LogSegment
{
  ...
  // dirty items 这部分都是对应的dir
  elist<CDir*>    dirty_dirfrags, new_dirfrags;
  elist<CInode*>  dirty_inodes;
  elist<CDentry*> dirty_dentries;

  elist<CInode*>  open_files;
  elist<CInode*>  dirty_parent_inodes;
  elist<CInode*>  dirty_dirfrag_dir;
  elist<CInode*>  dirty_dirfrag_nest;
  elist<CInode*>  dirty_dirfrag_dirfragtree;
  ...
}
```
我忘记代码是在哪里把脏的元数据加入到这个LogSegment的集合中了，后面想起来了再讲。
更新脏的元数据就是把这里的元数据根据他们所在的父目录，找到所有的父目录，再把这些父目录都更新到OSD上。注意，这里都是以目录为更新的基本单位。下面我们来看一下代码流程：
```
MDLog::try_expire() -> LogSegment::try_to_expire() -> CDir::commit() -> CDir::_commit() -> CDir::_omap_commit() -> objecter::mutate()
```
(1) LogSegment::try_to_expire函数主要是找出LogSegment类中记录元数据的坐在的目录Cdir
(2) 然后对每一个Cdir，把它目录下的内容都更新到OSD上
(3) 最后还会调用Objecter模块，这个模块也是OSDC中的，要把数据写到OSD上都要经过这个OSDC

- 4.2删除过期LogSegment
在脏元数据更新到OSD之后，就可以删除过期的LogSegment，这里还是删除的MDS中的Logsegment，主要就是从Logsegment的集合中删除过期的Logsegment。在MDS中的过期日志删除之后，还要删除OSD上的日志，为什么还要删除OSD上的日志，因为我们脏的元数据已经更新到OSD上了，所以OSD上的日志已经没有用了。
下面看一下函数调用：
```
MDLog::_trim_expired_segments() –> Journaler::write_head()
调用回调类  Journaler::C_WriteHead::finish() -> Journaler::_finish_write_head() -> Journaler::trim() -> filer::purge_range()
```
在删除完MDS中的过期Logsegment之后，会向OSD写一个日志文件的头（write_head）,这个头告诉OSD日志文件那些可以删除了，也就是OSD哪些日志可以删除了，这里是用一个回调类进行的，这个回调类是在OSD写头完之后，回复给MDS之后启动的。最后是调用trim，接着是调用filer模块来清理OSD上的过期日志。
