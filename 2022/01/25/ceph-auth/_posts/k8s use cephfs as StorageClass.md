---
title: k8s use cephfs as StorageClass
date: 2018-09-28 13:16:34
tags: k8s cephfs
---


# 问题
- 同事给了一个任务，habor想使用cephfs做后端存储，同时想使用storageclass动态分配pv

# 步骤
- 下载external-storage
```
# go get github.com/kubernetes-incubator/external-storage
```

## 创建cephfs命名空间
```
# cat namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
   name: cephfs
   labels:
     name: cephfs

# kubectl create –f namespace.yaml
```

## 部署external-storage
- 链接: https://github.com/kubernetes-incubator/external-storage/blob/master/ceph/cephfs/deploy/README.md

- 使用rbac role的 provider
```
# cd $GOPATH/src/github.com/kubernetes-incubator/external-storage/ceph/cephfs/deploy
# NAMESPACE=cephfs
# sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/*.yaml
# kubectl -n $NAMESPACE apply -f ./rbac –n cephfs
```
- 注：使用没有rbac部署的external-storage创建的pvc失败，后面再研究一下
这个会在cephfs命名空间下面创建一个pod，这个pod不会去调用cephfs创建pv

## 创建cephfs的secret
```
# cat ceph-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFCZDJIeGIyWmF6TlJBQTV4MnhvNDFuYU91VkhtUmJaTkEvOGc9PQ==

# kubectl create –f ceph-secret.yaml –n cephfs
```

## 创建StorageClass
```
# cat sc.yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cephfs
provisioner: ceph.com/cephfs
parameters:
    monitors: 10.125.224.26:6789
    adminId: admin
    adminSecretName: ceph-secret
adminSecretNamespace: "cephfs"

# kubectl create –f sc.yaml –n cephfs
```

## 创建pvc
```
# cat pvc_with_sc.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-3
  annotations:
    volume.beta.kubernetes.io/storage-class: "cephfs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi

# kubectl create –f pvc_with_sc.yaml –n cephfs
```

- 创建完pvc的时候会有pv创建，并且在cephfs中会有相应的目录创建
![aa](1.png)
在cephfs根目录下的volumes/kubernetes/kubernetes这个文件夹下

## 使用pvc创建pod
```
# cat pod_with_cephfs_pvc.yaml

kind: Pod
apiVersion: v1
metadata:
  name: test-pod1
spec:
  containers:
  - name: test-pod1
    image: docker.io/nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - name: pvc-3
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: pvc-3
      persistentVolumeClaim:
        claimName: pvc-3

# kubectl create –f pod_with_cephfs_pvc.yaml –n cephfs
```
![bb](2.png)
mount可以看到这个挂载关系

## 遇到的问题
之前pod一直创建不起来，打开了mds的日志看了一下，说client少FILE_LAYOUT_V2这个特性
解决办法传送门：
https://stackoverflow.com/questions/48026677/ceph-luminous-rbd-map-hangs-forever?rq=1
用内核挂载文件系统内核有版本要求，我目前是3.10，我升到了最新的4.18，然后就ok了。
