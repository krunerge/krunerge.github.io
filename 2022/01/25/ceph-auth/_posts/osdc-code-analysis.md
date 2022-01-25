---
title: 小甲师兄陪你一起看Ceph(osdc篇)
date: 2019-04-13 11:19:01
tags: Ceph Cephfs osdc
---

# 1. 深圳天气：小雨到阵雨到大雨
  小甲师兄有个喜好：喜欢下雨。每逢下雨天，不是诗兴大发，就是代码撸的飞起。再加上最近在优化rbd，小甲把之前分析的osdc
代码分享一下，这些都是小甲个人的理解，如果不对还望指正。

# 2.osdc
osdc其实是一个osd client模块的简称，在rbd和cephfs两个应用中都用到了，这个模块主要用来跟rados交互，这个模块里面完成了几个主要的功能：
- （1）：地址空间的转换：从rbd或者cephfs文件的一维地址空间转换到对象的三维地址空间，也简称为对象化
- （2）：objectcacher：一个object级别的缓存
- （3）：Crush算法定位osd：在转化为三维地址空间之后，就使用Crush算法进行对象的数据定位

本文主要是介绍前两点，小甲师兄14年刚开始接触ceph使用的是0.80.5版本（firefly），本文的分析是基于10.2.2（jewel），现在使用的是12.2.10（luminous），虽然版本跨度比较大，但是这部分代码还是比较稳定的，基本大的框架没有变化。这里以cephfs的文件读流程进行分析。

# 3.介绍一个文件的读
这里使用fuse进行挂载cephfs，因为用户态的客户端才会调用osdc，如果使用内核态的客户端的话，是不会经过osdc代码的，会走内核的一个osd client代码，估计功能跟osdc类似，不过内核的osd client的缓存是使用的内核中的一些缓存机制。
## 3.1 用户态文件请求
用户态文件请求的流程大致如下：
- （1）posix标准：open、read、write
- （2）系统调用
- （3）vfs虚拟机文件系统
- （4）fuse内核模块
- （5）fuse的用户态库
- （6）cephfs的用户态客户端代码
- （7）最终到达client::read（）或者client::write（）处理函数

## 3.2 client::read()
从client::read()起的流程图如下：
![a1](a1.png)

其中file_to_extents函数就是完成osdc的第一个功能：地址空间的转换

## 3.3 地址空间的转换
先贴一个对象条带分片图，也就是地址空间转换图，如下图所示：
![a2](a2.png)
这个对象分片图，是不是有点像raid0，对，就是将raid0中的磁盘换成了对象，并对对象进行了条带分片。
- 按上图介绍几个概念
（1）蓝色柱状代表一个个rados底层对象，默认为4M
（2）绿色的su代表条带单元
（3）红色的stripe代表一个个条带
（4）objectset代表对象组，一般一个对象组属于同一个文件（cephfs）

这里要说明一点，Ceph系统默认没有开启对象条带分片

- 再介绍几个变量
（1）object_size：对象的大小，就是rados底层对象的大小，一般默认是4M
（2）su：对象分片大小，以上面的图为例就是4/3M
（3）stripe_count：条带宽度，也就是一个strip跨多少个对象，也就是一个objectset中对象的个数，以上面的图为例，值为3
（4）stripes_per_object：一个对象包含的对象分片数，以上图为例值为3

- file_to_extent
 file_to_extent函数就是以raid0的思想来分片，一个个条带连起来可以看成是逻辑上面连续的，相当于线性的一维地址空间。现在要通过file_to_extent函数把一维坐标转化成三维坐标(objectset，stripeno，stripepos)，这三维坐标分别表示哪一个objectset，哪一个条带，条带中的哪一个对象分片。小甲以一个读操作来分析，示例分析如下：
![a3](a3.png)
这里假设一个对象大小是3M，一个对象分片大小是1M，假设我们要读的文件占用两个objectset，占用6个rados的对象，18个对象分片，小甲这里要读取这个文件的对象分片的序号是从第1到第6，也就是文件的2M~7M的范围，相应的变量如下所示：
```
offset = 1M 表示读偏移量
len = 6M 表示要读取的大小
su = 1M
object_size = 3M
stripe_count = 3
stripes_per_object = 3
```
可以看到上面的地址空间已经从一维转化成了三维：比如读取su1
一维地址空间：(offset, len) ==>（1M，1M）
三维地址空间：(objectset，stripeno，stripepos) ==> （objectset0，stripe0，object1）

- 对象名的组成
这里的对象指的是rados底层的对象，也就是使用filestore时，xfs上面一个个4M的小文件，那这个文件的名字是怎么组成的呢，小甲顺便也分析了一下。
这里先说明一个类ObjectExtent，这个是用来保存一个对象内的分片的信息，注意这个类不保存内容，只是保存分片的位置信息：偏移和长度。分片的结果会保存在一个map中
```
map<object_t,vector<ObjectExtent> >  object_extent
```
map的key是一个对象的名字，这个名字就是存在磁盘上面每一个对象对应文件的文件名的前缀，比如，磁盘上面一个对象的文件名为:
```
10000000000.00000000__head_F0B56F30__1
```
其中10000000000.00000000这个由点号隔成了两部分，点号前面的10000000000是一个文件的元数据inode号，后面00000000是文件内容分配所在的对象号，这个对象号有点特殊代码中叫做objectno，我们来具体讨论一下。
首先每一个文件都是可以分片成对象，按照分片算法，这里分片所对应的对象应该都是从0开始标号的，那这样对象的名字不就重复了吗，不急，我们每一个文件的inode号是一个唯一的这个inode号是由mds中的inodetable维护的，保证唯一，那就可以使用文件的元数据inode号和刚刚可以重复利用的objectno就组成了上面我们看到的由点号分开的对象名前缀，其实这个感觉就像我们的网络，在openstack中我们可以给每一个租户分配一个网络，网络里面可以自己划分子网，不同租户的子网的网段是可以重复的，因为他们就相当于一个局域网，可以重复利用，这里面的objectno就相当于局域网段，可以重复利用，但是和inode号合在一起就是一个全局唯一的，就是存储在磁盘上面的前缀。

## 3.4 对象分片与ObjectExtent
对象分片跟objectextent的对应关系有点复杂, 听小甲慢慢分析。
- 1.因为要使用osdc就要用用户态的客户端，就是使用fuse，但是内核态的fuse的模块对读写数据大小是进行了限制，一次写最多是4K，而读是128K，也就是说我们如果像我们列子中要读取文件中1M到6M之内的内容不是一次传过来的，而是在fuse内核模块那边就已经分成了一个个128K的读，估计fuse用户态的库那边有一个线程池，所以每一次的128K都是并发由不同的线程进行处理。

- 2.因为上面fuse内核已经给我们的请求进行了分片，传过来的是一个个128K的读长度请求，对于我们例子中的情况，一个对象分片su是1M的情形，我们来细细的分析一下：
 2.1.首先传来第一个128K的文件长度的读，那就是从偏移量为1M的开始读取长度为128K的文件数据，我们该怎么映射分片呢?
 ```
 offset = 1M
 len = 128K
 ```
 步骤如下：
 - 第一步计算出偏移量的位置信息，这个位置信息我们要把它从一维的转化为三维的坐标，计算过程如下：
 ```
 blockno = offset/su =1M/1M =1         块号，也就是分片号就是su1
 stripeno = blockno /stripe_count = 1/3 =0    条带号，表示一个条带stripe0
 stripepos  = blockno%stripe_count = 1%3 = 1  条带内偏移，就是在条带内的第二个对象上面
 objectsetno = stripeno / stripes_per_object=0/3 =0   对象set号，表示objectset0
 objectno = objectsetno*stripe_count + stripepos= 0*3+1对象号，就是分片所在的哪个对象
 ```
 这样我们就把以为的坐标转化为三维的坐标: （su1） --> (objectset0, stripe0, stripepos=1)
![a4](a4.png)

 - 第二步修改分配内的偏移量
```
 block_start = (stripeno % stripes_per_object) * su = (0%3)*1M =0 , 这个表示的是文件操作偏移量在所对应的对象中的偏移
 block_off = offset % su = 1M %1M =0,这个表示的是文件操作偏移量在所对应的对象su中的偏移
 x_offset = block_start + block_off = 0 + 0,这个表示对象分片的偏移实际地址
 max = su - block_off=1M -0 = 1M,这个分片还剩的最大容量
 x_len = min（len，max),分片内的偏移量的最小值
```
上面公式太复杂，我们下面以几个列子来寻求普遍情况:
例子1：是上面的128K，从su1边界开始
![a5](a5.png)
可以看出我们的
```
x_offset = 0
x_len=128k
然后生成一个objectextent，里面的offset=x_offset = 0，length=x_len=128k
```

例子二：如下情景
![a6](a6.png)
可以看出我们的：
```
x_offset = 64k
x_len=128k
然后生成一个objectextent，里面的offset=x_offset = 64k，length=x_len=128k
```

例子三：如下情景
![a7](a7.png)
虽然我们知道在我们假设的su为1M，fuse一次读分片是128K的情况下是不会出现这种情况的，当时我们可以调整su的大小，调整对象的大小，这样的情况就可以达到了。
这里我们假设去掉fuse读分片128k的限制，我们假设读了1M的数据，并且我们su为1M，现在我们要读上面所有绿色的部分。对于前面三个分片，我们可以知道，第一个分片在object1的第一个su中，不是一个整的分片，第二个分片是在object2中的第一个su中，完整的一个分片，第三个分片是在object0中的第二个su，是一个完整的分片，最后一个分片是在object1的第二个su中，我们可以知道，对于前三个分片在，分片结果集map<object_t,vector<ObjectExtent> > object_extent会存放三个对象，假设我们还没有映射最后一个分片我们只是映射了三个分片，此时，object_extent下的状态是如下：
![a8](a8.png)
那对于第四个分片呢，会在object1下面在生成一个objectextent，这个里面的分片是不全的，现在有一个问题来了是不是一个su对应一个objectextent？
答案是的，一个su对应一个objectextent，可能这个objectextent没有su那么大的范围，即len(objectextent)<=len(su)，而且在objectextent结构中有一个vector<pair<uint64_t,uint64_t> >buffer_extents，这个是用来记录此分片相对于我们要读的偏移位置的偏移量，我们画个图解释一下。
![a9](a9.png)
buffer_extents里面的key是指的蓝色小格相对于offset的偏移，不过小甲在撸完代码之后，感觉基本上一个objectextent里面的buffer_extents只会存在一个pair，不会存在多个pair，一般要么就是一个su或者就是小于su。

## 3.5 对象分片之后跟objectcacher中的缓存关联
从上面的file_to_extent中得到映射后的结果集object_extent，这个是一个map，首先遍历取出所有的objectextent放在一个vector中。然后下面的在readx函数中去读每一个映射后分片objectextent上面的内容，因为objectextent结构中包含了要读对象的名字和要读内容在这个对象中的迁移位置，所以就可以对每一个objectextent进行并发读取。但是在读的时候由于使用objectcacher缓存，我们可以现从缓存中查找看看我们要找的objectextent范围的内容是不是在缓存中，缓存命中在直接取出，不命中就需要去rados读。
### 映射关系
这里就有一个问题，就是objectextent如何跟bufferhead映射起来？接下来我们分析一下。
 这主要是一个函数map_read，这个函数就是做对一个objectextent进行映射，这里面的映射也很有趣，我们来看一下，首先说明一下bufferhead是一个对象内部的片段，毫无规律的。
 object和bufferhead之间的关系如下，这里的object严格说不是rados的object，是osdc缓存中的对象，其实可以说是内存中与磁盘对应的数据结构。
![a10](a10.png)
上图中一个object中有三个bufferhead
下面分析几种情景来说明object、bufferhead和objectextent之间的关系
- （1）objectextent的范围在现有的bufferhead之后
![a11](a11.png)
 这种情况下，会生成一个bufferhead，包含objectextent的范围，加入一个bufferhead之后变成这样:
![a12](a12.png)

- (2) objectextent的范围在某一个bufferhead之内
![a13](a13.png)
 像上面这种情况，表示一部分已有映射，其实上面这种情况还有一部分在外面，最后变成这样
![a14](a14.png)

- (3) objectextent的范围在间隔段之间
![a15](a15.png)
像上面这种情况，objectextent中间某一部分已经映射到对象的某部分，最后映射的结果如下
![a16](a16.png)
 一般一个objectextent会对应到多个bufferhead上面

 ### 缓存状态
 这里我们只是做了映射关系，那如何看这些bufferhead中是不是都缓存了对象的数据呢，这个就是根据bufferhead的状态来进行，bufferhead有很多状态，我们来看一下
![a17](a17.png)
 bufferhead有这几种状态，根据代码中的意识是当状态为clean，dirty，tx和zero时，可以认为缓存中是有对象数据的

 ## 3.6 读操作引起的发送到rados上面的操作序列
 读操作请求一般会经过这几个过程
（1）首先去mds上面找到文件所对应的元数据，如果mds没有缓存inode，就要到osd上面去取元数据，此操作其实就时打开文件open
（2）去到元数据之后，inode里面包含了文件数据在osd上面的一些位置信息file_layout_t，这个里面主要是包含文件分片的一些参数，还有文件所在的池信息
（3）向osd发起了一个stat op的操作请求，返回文件的属性
（4）然后会发送一个create op的操作请求
（5）后面才会从fuse内核接受一个个128k的读请求

## 3.7 我们拿一个128K的请求来看
假设我们缓存中刚开始是空的，好那我们在经过map_read之后我们我们流程图如下
- （1）第一次读缓存未命中到osd上去取
![a18](a18.png)
因为缓存未命中，所以在bh_read中发起一个到osd去对象数据的op操作
-  (2) 客户端接收到osd读请求发送回来的响应
![a19](a19.png)
这里又开始进行第二次读，这次读就能在缓存中命中
-  (3) 第二次读流程，命中缓存
![a20](a20.png)

# 4. 结尾
小甲这一期讲的东西有点多，这里面还是很复杂的，读者需要静下心来结合代码好好分析。
