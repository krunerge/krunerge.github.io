---
title: nova Pike changelog分析
date: 2018-09-26 16:14:27
tags: openstack nova
---

# **nova-16.0.1修改**
- **Enable custom certificates for keystone communication**
  允许keystone通信的自定义认证
- **Add @targets\_cell for live\_migrate\_instance method in conductor**
  live_migrate_instance增加了一个装饰器
- **Set error state after failed evacuation**
  疏散失败的虚拟机设置成"error"状态，原来是"accepted"状态，如果这个状态的话在nova-compute重启之后会有问题
- **Functional test for regression bug #1713783**
  增加功能测试，在没有机器可以疏散的时候设置虚拟机状态为"error"
- **Call terminate\_connection when shelve\_offloading**
  增加虚拟机断开连接卷的处理，因为虚拟机在做了shelve操作之后进行unshelve的时候不一定还会调度到相同的机器上
- **Handle keypair not found from metadata server using cells**
  修复bug https://bugs.launchpad.net/nova/+bug/1592167
  在使用cell的时候，如果删除了创建虚拟机时的keypair，那么在检索虚拟机的时候会报400错误
- **Target context when setting instance to ERROR when over quota**
  通知虚拟机所在的cell，当虚拟机设置为error的时候
- **Move hash ring initialization to init\_host() for ironic**
  把hash环的初始化移到了ironic的init_host函数中
- **Remove dest node allocation if evacuate MoveClaim fails**
  当疏散失败之后，需要移除目标机器的资源分配量，因为资源tracker不会自动去释放
- **De-duplicate two delete\_allocation\_for\_\* methods**
  抽取迁移和疏散函数内部的公共代码部分，属于代码重构
- **Add a test to make sure failed evacuate cleans up dest allocation**
  增加功能测试来验证在疏散操作失败之后会清理目的机器分配资源的清空
- **Add recreate test for evacuate claim failure**
  增加重建功能测试，在目的主机MoveClaim失败之后上面的资源清空失败
- **Create allocations against forced dest host during evacuate**
  在疏散操作期间，如果指定了目的主机，则忽略掉调度select_destinations()过程
- **Refactor out claim\_resources\_on\_destination into a utility**
  把claim_resources_on_destination函数提取出来放入到scheduler的工具文件中，属于代码重构
- **Add recreate test for forced host evacuate not setting dest allocations**
  指定目标主机的疏散的功能测试
- **Provide hints when nova-manage db sync fails to sync cell0**
  增加数据库同步cell0错误提示
- **Ensure instance mapping is updated in case of quota recheck fails**


# **nova-16.0.2修改**
