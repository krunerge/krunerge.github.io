---
title: 听说你的harbor不能使用s3
date: 2019-03-16 19:06:53
tags: harbor docker ceph s3
---

# harbor
harbor主要是用来存储容器镜像的开源项目，也就是所谓的容器镜像仓库，跟OpenStack的glance功能类似，不同的是后者存放的是虚拟机镜像。
harbor除了可以存储容器镜像，还可以存储chart，chart是容器环境下的应用打包格式，类似于centos下的rpm包，ubuntu下的debian包。
下图是harbor的架构图
![a](a.png)
这两种资源都需要放在存储介质上，镜像和chart可以分开用不同的存储，也可以使用相同的存储后端，这里主要讨论一下镜像的存储。
从上图可以看到harbor对镜像的实际存储使用的是docker原生的registry，那研究harbor的镜像存储，就是研究docker registry的镜像存储。一般都存放在本地文件系统上，

即后端存储driver使用filesystem，由于harbor是跑在容器里的，此driver根据具体实现又可分为以下三种情况：
- (1)服务器本地存储：这种方式就是把服务器本地文件目录挂载到容器中做为harbor存储镜像的位置
- (2)ceph rbd pv:这种方式首先通过k8s的pvc申请一个ceph rbd卷，然后把rbd卷mount到服务器，最后再把这个目录挂载到容器中
- (3)ceph cephfs pv:这种方式直接使用cephfs功能，通过pvc申请一个cephfs中的一个目录，然后mount到服务器，最后在把这个目录挂载到容器中

简单说明一下这三种：第一种不是共享存储，是harbor高可用的拦路虎；第二种ceph rbd卷出于数据一致性的考虑，不能同时挂载在对各宿主机上，也是harbor高可用的拦路虎；第三种cephfs是共享存储，通过文件系统是可以进行多挂载，同时读写，但是由于cephfs storageclass driver不是官方指定的，可能driver质量还不够高，同时cephfs本身也质疑比较多（不过社区一致在对cephfs投入很大）
好像没有一种比较十全十美的，那还有其他的driver吗？


# ceph rgw可用否
ceph一共就三种应用，块、文件、对象，前两种都试过了，那ceph的rgw对象存储可以使用不，看了一下docker registry中的driver是用s3，ceph的rgw是支持s3的，那应该可以？！不过网上搜了一下好多人说，都有问题，一直卡主retry，比如一下连接：https://github.com/docker/distribution/issues/2189，
![b](b.png)
小甲以为这个问题应该解决了，毕竟这是2017的issue，先实验试一下

# docker registry接ceph rgw s3实验
- 版本：
```
harbor
docker registry(distribution)  2.6.2
aws-sdk-go
```
docker registry后来v2版改名叫distribution，上面表是实验使用的代码版本
- 配置：
distribution配置
```
s3:
  region: us-east-1
  bucket: gkkbucket1
  accesskey: tstackqwer666
  secretkey: tstackqwer666
  regionendpoint: http://10.85.46.87:8080
  secure: false
```
10.85.46.87:8080是ceph rgw的地址
docker的/etc/docker/daemon.json配置如下：
```
{
  "insecure-registries": ["192.168.127.1:5000"]
}
```
192.168.127.1:5000是distribution服务的地址，也就是让docker使用私有镜像仓库

docker tag scratch:latest 192.168.127.1:5000/scratch:1.11

# 实验结果：registry接s3有问题
运行distribution，然后docker push私有镜像，实验表明确实还存在问题，一致在retry
假装有图。
网上没有解决办法，那就手撸一下。

# 调试及分析
- 1.首先制作一个只有一层的镜像，要不然distribution调试的时候请求会很多，眼花缭乱，不要给自己添麻烦，
这里使用如下命令制作一个只有一层的镜像
```
tar cv --files-from /dev/null | docker import - scratch
```
- 2.设置tag
```
docker tag scratch:latest 192.168.127.1:5000/scratch:1.11
```
通过调试发现docker registry是先上传数据，最后再上传manifest，通过调试发现
![c](c.png)
```
192.168.127.133 - - [23/Mar/2019:20:30:45 +0800] "PUT /v2/docker.io/alpine/blobs/uploads/4f495fbb-876f-43b7-8aad-166701220985?_state=BbHh5XVtlX93j2I3vYVRWNluZ96_3wIhkMKd-9W3YeB7Ik5hbWUiOiJkb2NrZXIuaW8vYWxwaW5lIiwiVVVJRCI6IjRmNDk1ZmJiLTg3NmYtNDNiNy04YWFkLTE2NjcwMTIyMDk4NSIsIk9mZnNldCI6Mjc1NDcyOCwiU3RhcnRlZEF0IjoiMjAxOS0wMy0yM1QxMjozMDozNVoifQ%3D%3D&digest=sha256%3A6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0 HTTP/1.1" 201 0 "" "docker/1.13.1 go/go1.10.3 kernel/3.10.0-327.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))"
time="2019-03-23T20:30:47.8196001+08:00" level=debug msg="authorizing request" environment=development go.version=go1.11.5 http.request.host="192.168.127.1:5000" http.request.id=5ab88eed-272e-4c63-ac4e-4395284f1647 http.request.method=HEAD http.request.remoteaddr="192.168.127.133:34417" http.request.uri="/v2/docker.io/alpine/blobs/sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" http.request.useragent="docker/1.13.1 go/go1.10.3 kernel/3.10.0-327.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))" instance.id=ea6a2d2d-5f9e-4a14-8e66-f373a70a9298 service=registry vars.digest="sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" vars.name="docker.io/alpine" version="v2.6.2+unknown"
time="2019-03-23T20:30:47.8196001+08:00" level=debug msg=GetBlob environment=development go.version=go1.11.5 http.request.host="192.168.127.1:5000" http.request.id=5ab88eed-272e-4c63-ac4e-4395284f1647 http.request.method=HEAD http.request.remoteaddr="192.168.127.133:34417" http.request.uri="/v2/docker.io/alpine/blobs/sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" http.request.useragent="docker/1.13.1 go/go1.10.3 kernel/3.10.0-327.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))" instance.id=ea6a2d2d-5f9e-4a14-8e66-f373a70a9298 service=registry vars.digest="sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" vars.name="docker.io/alpine" version="v2.6.2+unknown"
time="2019-03-23T20:30:47.8196001+08:00" level=debug msg="s3aws.URLFor(\"/docker/registry/v2/blobs/sha256/6c/6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0/data\")" environment=development go.version=go1.11.5 http.request.host="192.168.127.1:5000" http.request.id=5ab88eed-272e-4c63-ac4e-4395284f1647 http.request.method=HEAD http.request.remoteaddr="192.168.127.133:34417" http.request.uri="/v2/docker.io/alpine/blobs/sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" http.request.useragent="docker/1.13.1 go/go1.10.3 kernel/3.10.0-327.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))" instance.id=ea6a2d2d-5f9e-4a14-8e66-f373a70a9298 service=registry trace.duration=0s trace.file="D:/Go/src/github.com/docker/distribution/registry/storage/driver/base/base.go" trace.func="github.com/docker/distribution/registry/storage/driver/base.(*Base).URLFor" trace.id=83cf9b9e-46a4-43a3-9db2-095ece7dda28 trace.line=189 vars.digest="sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" vars.name="docker.io/alpine" version="v2.6.2+unknown"
time="2019-03-23T20:30:47.8206001+08:00" level=info msg="response completed" environment=development go.version=go1.11.5 http.request.host="192.168.127.1:5000" http.request.id=5ab88eed-272e-4c63-ac4e-4395284f1647 http.request.method=HEAD http.request.remoteaddr="192.168.127.133:34417" http.request.uri="/v2/docker.io/alpine/blobs/sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0" http.request.useragent="docker/1.13.1 go/go1.10.3 kernel/3.10.0-327.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))" http.response.contenttype="application/octet-stream" http.response.duration=2ms http.response.status=307 http.response.written=0 instance.id=ea6a2d2d-5f9e-4a14-8e66-f373a70a9298 service=registry version="v2.6.2+unknown"
192.168.127.133 - - [23/Mar/2019:20:30:47 +0800] "HEAD /v2/docker.io/alpine/blobs/sha256:6c40cc604d8e4c121adcb6b0bfe8bb038815c350980090e74aa5a6423f8f82c0 HTTP/1.1" 307 0 "" "docker/1.13.1 go/go1.10.3 kernel/3.10.0-327.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/1.13.1 \\(linux\\))"
```
在这里卡主了，上面有一个url从定向,
ceph rgw的日志如下：
![d](d.png)
首先请求包认证错误，怎么会认证错误呢，再查看请求头发现这请求奇怪
![e](e.png)
rgw请求的s3 v4认证怎么在这个QUERY_STRING字段，这个字段一般是url后面的查询条件，小甲思考3秒，可能是url重定向导致，屡一下可能是这样
- 1.docker向distribution发送了一个head请求
- 2.distribution接收到请求之后，进行了url重定向，把signure认证放在了url查询部分，放在location中返回docker
- 3.docker根据返回请求中location中的重定向的url，也就是上面ceph rgw的地址，然后发送请求，可是docker没有调用任何s3的sdk，所以不会进行s3的signure算法，直接head发给了ceph rgw
- 4.ceph rgw接收到head请求，还是根据s3的v4认证，所以报错了。

后面调试代码发现问题确实是这样。

# 修改方法
那如何修改呢，其实修改很简单，只要把url直接抛异常就可以，这个filesystem也是这样处理的。小甲本来的想法是新建一个结构体，继承现有的s3 driver，然后重载URLFor函数就可以，但是发现s3的driver是包外不可见，这就无法继承，最后通过增加一个配置参数，是否是使用的ceph s3还是aws s3，在使用ceph s3的时候直接抛异常退出URLFor函数。

#等等还没结束
docker push和pull现在都可以正常了，但是镜像的删除也有问题，那镜像是怎么删除的呢？
- 1.调用api删除
- 2.distribution 运行garbage-collect命令
发现有问题，ceph接受不到请求，调试了好久发现是distribution中vendor目录下的aws-sdk-go版本比较老，小甲用较高的版本就可以了
这下应该结束了吧，no no no

# distribution master问题
小甲又下载了distribution的master分支试了一下，发现前面的修改之后docker push/pull都ok，删除换了高版本的aws-sdk-go还是有问题，什么问题呢？
- 1.空指针异常，What？
![f](f.png)
在doWalk函数中出现了空指针，这个函数在distribution 2.6.2中还没有
- 2.代码分析从ceph返回来的
ListObjectsV2Output这个对象的KeyCount这个成员是一个空指针，而代码中使用这个进行了运算导致了错误，看来distribution确实没有验证ceph
- 3.修改也比较简单，通过之前加的一个ceph配置对ceph的请求情况做一下特殊处理就可以了。


# 修改代码后重新编译
  编译修改后的代码成二进制可执行文件

# 重新制作harbor-registry镜像
 使用dockerfile生成新的镜像
 ```
 FROM  test:v1.7.0

COPY ./registry /usr/bin
EXPOSE 5000
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/etc/registry/config.yml"]
 ```

# chart配置修改
修改harbor/values.yaml文件
```
imageChartStorage:
    # Specify the type of storage: "filesystem", "azure", "gcs", "s3", "swift",
    # "oss" and fill the information needed in the corresponding section. The type
    # must be "filesystem" if you want to use persistent volumes for registry
    # and chartmuseum
    type: s3
    filesystem:
      rootdirectory: /var/lib/registry
      #maxthreads: 100
    azure:
      accountname: accountname
      accountkey: base64encodedaccountkey
      container: containername
      #realm: core.windows.net
    gcs:
      bucket: bucketname
      # TODO: support the keyfile of gcs
      #keyfile: /path/to/keyfile
      #rootdirectory: /gcs/object/name/prefix
      #chunksize: "5242880"
    s3:
      region: us-east-1
      bucket: gkkbucket2
      accesskey: tstackqwer666
      secretkey: tstackqwer666
      regionendpoint: http://10.85.46.87:8080
      secure: false
      cephcompatible: true
      #encrypt: false
      #keyid: mykeyid
      #v4auth: true
      #chunksize: "5242880"
      #rootdirectory: /s3/object/name/prefix
      #storageclass: STANDARD
```

# 参考链接
http://dockone.io/article/93
