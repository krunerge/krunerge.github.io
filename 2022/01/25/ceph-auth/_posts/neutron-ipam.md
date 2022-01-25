---
title: neutron的ipam还可以这么玩
date: 2019-01-24 11:35:41
tags: openstack neutron
---
![c](c.png)
neutron作为openstack的核心组件用来管理网络，比如network，subnet和port核心资源，lbaas，fwaas等扩展资源，但是对于用户来说最基础的是ip，以及这个ip所在的网络，这部分功能是由neutron的ipam管理。

# 哪些地方需要ip
————————————————————————————————————————————————————————————————————————————
neutron哪些地方用需要ip？too many，比如说虚拟机的网卡，路由器的外部网关、内部接口，浮动ip，dhcp port，ha路由器的管理口，其他组件lbaas，manila等都需要ip，这些ip的分配其实在neutron看来都是创建port资源，只是各种device owner不同，都需要通过neutron的ipam分配和管理。

# IPAM是个啥
————————————————————————————————————————————————————————————————————————————
IPAM（IP Address Management）,也可以叫做ip地址管理子系统。L版neutron的ipam是一个分水岭，L版之前ip的管理只能使用neutron本身的数据库，分配和回收都是通过数据库来实现。L版开始，社区对IPAM进行了重构，把ip的分配和释放以及子网的增删改查作为标准的接口提取出来，使用driver的方式来接入各种拥有ip和子网管理功能的系统，这样就实现了IPAM可插拔实现，接入用户一些现有的ip管理系统，而对neutron其他功能基本没有影响，L版之前使用数据库实现的ip管理作为了IPAM一种默认的driver。
k版neutron的ipam目录下结构
```
[root@con01 ipam(keystone_admin)]# ll
total 32
-rw-r--r-- 1 root root  7428 Aug 21  2015 __init__.py
-rw-r--r-- 1 root root  4124 Aug 21  2015 driver.py
-rw-r--r-- 1 root root 16282 Aug 21  2015 subnet_alloc.py
```
P版neutron的ipam目录下结构
```
[root@con01 /usr/lib/python2.7/site-packages/neutron/ipam]# ll
total 52
-rw-r--r-- 1 root root     0 Aug 31 16:29 __init__.py
-rw-r--r-- 1 root root  6733 Aug 31 16:29 driver.py
drwxr-xr-x 4 root root  4096 Jan 24 14:31 drivers
-rw-r--r-- 1 root root  2916 Aug 31 16:29 exceptions.py
-rw-r--r-- 1 root root 11534 Aug 31 16:29 requests.py
-rw-r--r-- 1 root root 19006 Aug 31 16:29 subnet_alloc.py
-rw-r--r-- 1 root root  2616 Aug 31 16:29 utils.py
```
其中drivers目录下面就是各个driver的实现，其他文件定义了ip和子网管理相关的标准接口，各个driver根据后端管理平台具体实现标准接口
```
[root@con01 ~]# ll /usr/lib/python2.7/site-packages/neutron/ipam/drivers/
total 16
-rw-r--r--. 1 root root    0 Oct 31 06:27 __init__.py
drwxr-xr-x. 2 root root 4096 Dec 24 16:09 neutrondb_ipam
```
从上图中的driver文件夹的名字也可以看出，默认的driver就是使用数据库来管理ip和子网。

# 默认driver分析
————————————————————————————————————————————————————————————————————————————
![d](d.png)
架构图
从架构图可以看出ipam主要和ML2 plugin和Ipam db交互，那我们下面主要分析一下怎么交流的，由于ipam主要管理子网和port，网络ipam不管理，主要分析一下port和子网的创建
- port的创建流程：
  1. neutron server接受创建port的请求之后，到达ML2 plugin模块进行处理，ML2 plugin主要处理网络、子网和port资源
  2. ML2 plugin把请求交给Ipam处理，Ipam主要是用来给port分配一个可用的ip或者用户指定ip，对于具体使用什么分配算法根据driver不同而不同，默认的neutronDB driver使用的是如下算法：
  首先从Ipam相关的数据库取出已经使用的ip，然后列举子网上面所有的ip，取个差集，就是可用的ip池，然后通过一个随机ip窗口在可用ip池中获取一个可用的ip。
  3. 通过Ipam获取完ip之后，创建port数据结构加入到neutron的db中，这里Ipam db和neutron db其实都是在neutron数据库中，小甲之所以这么叫，主要是区分它们的作用，Ipam db主要是用来管理ip分配、可用池等，neutron db主要是为了记录port信息，ip只是它的一个属性。
  4. 后面会把请求交给type manager和Mechainsm manager创建相应的底层资源
![a](a.png)

- 子网的创建流程：
  1. neutron server接受创建子网的请求之后，还是ML2 plugin模块处理
  2. ML2 plugin把请求交给Ipam处理，Ipam主要是在Ipam db中创建子网的数据模型实例，记录子网信息，比如子网的默认网关、dhcp的ip、有效的ip地址
  3. 在Ipam创建完数据库实例之后，会在neutron db创建子网资源，跟port一样，Ipam数据库中的子网只要是用来分配ip的，neutron数据库中主要是neutron层记录子网，比如我们进程用到neutron subnet-list就是这个neutron数据库获取的。
  4. 后面会把请求交给type manager和Mechainsm manager创建相应的底层资源。
  5. 子网的创建，会触发创建这个子网上面的dhcp port，创建流程跟上面port一样，然后dhcp agent会创建相应的资源。
![b](b.png)

# 如何写自己的Ipam driver
（1）知道要实现基类数据模型：
- 请求工厂基类，主要创建ip分配请求和子网创建请求：
1. SubnetRequestFactory：用于子网创建请求
2. AddressRequestFactory： 用于ip分配请求
各个driver可以根据需求通过不同工厂创建不通的请求
- ip和子网操作基类模型如下图：
![f](f.png)
各个driver根据自己ip系统的不同而不同
（2）driver请求调用ip系统流程
这个主要看个人代码结构，可以直接调用后端ip系统的restful api，也可以逻辑清楚的划分各个层，Tstack划分了controller、manager和对象映射api层

# Ipam扩展有什么好处
————————————————————————————————————————————————————————————————————————————
那么问题来了，ipam driver有什么好处，以我们Tstack为例
1. 我们Ip系统比neutron默认的功能更强大丰富，默认的ipam有些场景不能满足我们需求
2. Tstack借助Ipam driver可以使Ip系统统一管理整个平台的网络资源
3. 上层应用可以接入Ip系统实现运维管理
4. ...

# 遇到的问题
————————————————————————————————————————————————————————————————————————————
在实现IPAM驱动的过程中，遇到过很多问题，并一一进行了解决，大致列举一两个问题以引起大家注意
- 1.neutron数据模型跟自研ip系统模型不匹配，比如少一些模型，或者一些模型不完全对应，会导致IPAM在跟自研ip系统交互的时候出现问题。
- 2.业务信息的处理，自研ip系统的定位可能不是底层组件，会带有一些上层业务信息，而neutron没有这些业务信息，比如ip系统会根据业务分网络类型，基础网、私有网等，而neutron不分这么细
- 3.neutron多入口访问的统一，restful api，命令行和dashboard多个入口的访问要保证的结果一样
- 4.IPAM和ip系统的两部分的配合处理需要处理好并发访问处理
