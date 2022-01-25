---
title: how to change horizon chinese
date: 2018-12-15 13:17:08
tags: openstack horizon
---

# 问题描述
最近笔者想修改一下openstack dashboard上面中文，比如我想把"主机聚合"改成"主机集群"，笔者的openstack的版本是Pike，虽然这个问题网上有很多说明，但是不能解决我的全部问题

# 问题分析
网上都说在/usr/share/openstack-dashboard/openstack_dashboard/locale/zh_CN/LC_MESSAGES这个目录下会有一个po文件，只要把里面的相应中文改成你想要的就可以了，但是Pike版环境上这个目录下面没有这po文件，只有一个mo文件
![aa](1.png)
而且还是二进制文件，打开看不出啥一对乱码
然后笔者想到的是去github的原理里面找po文件
https://sourcegraph.com/github.com/openstack/horizon@master/-/blob/horizon/locale/zh_CN/LC_MESSAGES/django.po
然而。。。。
我在这个文件里面搜索竟然没有找到我要求的中文，什么鬼，那到底在哪里
然后很无奈的全局搜索了一下，最后竟然在上面mo二进制文件中有match，那就说有，那为什么源码的po文件没有，估计应该是编译的时候加入一些

# 问题解决
那问题就是如果把mo反编译会po，然后修改po，最后在编译会mo
mo到po
```
msgunfmt xxx.mo -o xxx.po
```
po到mo
```
msgfmt -o xxx.mo xxx.po
```
