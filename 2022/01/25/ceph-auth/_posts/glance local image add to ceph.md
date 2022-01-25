---
title: glance本地镜像补齐到ceph
date: 2018-09-27 17:11:30
tags: openstack glance
---

# 问题
  由于同事的openstack部署脚本每次都会先部署本地存储的glance，然后上传一个镜像，测试创建虚拟机功能，然后在glance配置ceph后端存储，这会导致之前上传上去的镜像不能使用来创建虚拟机。

# 解决办法
通过补齐镜像到ceph来实现，有一下几个步骤：
1.在安装glance的机器上面找到镜像的本地存储
```
# cd /var/lib/glance/images/
# ll
total 12892
-rw-r----- 1 glance glance 13200896 Sep 25 09:32 09913ab8-5b38-4fcb-880a-4efa23bb229a
```

2.上传镜像到ceph
```
rbd import 09913ab8-5b38-4fcb-880a-4efa23bb229a  images/09913ab8-5b38-4fcb-880a-4efa23bb229a
```

3.创建镜像快照snap
```
rbd snap create --image images/09913ab8-5b38-4fcb-880a-4efa23bb229a --snap snap images/09913ab8-5b38-4fcb-880a-4efa23bb229a@snap
```

4.修改glance数据库
glance数据库中有一个image_locations记录了glance的位置，是本地存储file，还是ceph rbd存储
```
update image_locations set value="rbd://dc21b052-eae7-46bb-a90d-4818a029a99c/images/09913ab8-5b38-4fcb-880a-4efa23bb229a/snap" where image_id = "09913ab8-5b38-4fcb-880a-4efa23bb229a";
```

OK，大功告成
