# Kubernetes(七) - Volume
#w-blog博客/kube

![](Kubernetes(%E4%B8%83)%20-%20Volume/1B88873A-A973-4B22-A7BB-945B4E30394E.png)

Docker是无状态的不管被销毁多少次都会恢复到最初的状态,但是这就意味着在程序过程中产生的配置也好文件也好会丢失,对于Docker我们经常会使用磁盘挂载的方式来保存一些重要的内容,比如运行在Docker下的数据库的源数据,比如程序的日志文件等,在K8S中也提供同样的配置方式

> PS: 磁盘使用中1.8 和 1.9存在差异,1.8需要创建PersistentVolume在创建之后才能创建PersistentVolumeClaim,1.9之后只需要创建PersistentVolumeClaim就可以了  

附上:

喵了个咪的博客:[w-blog.cn](w-blog.cn)
Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  

## 1.本地磁盘

```
> vim local-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-1
  labels:
    type: local
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/data/pv-1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pv-claim
  labels:
    app: redis
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi

> kubectl create -f local-pv.yaml
persistentvolume "local-pv-1" created
persistentvolumeclaim "mysql-pv-claim" created
```

![](Kubernetes(%E4%B8%83)%20-%20Volume/2964A0D8-7EC4-4F70-B428-51BD370A3A04.png)


![](Kubernetes(%E4%B8%83)%20-%20Volume/2897145C-258E-423E-AE40-DE04E298B645.png)

然后我们就可以对对进行进行挂载了

```
> vim volume-local.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-local-pod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:                         # 磁盘挂载
      - name: redis-pv-claim
        mountPath: "/etc/redis"
  volumes:                                  # 磁盘挂载别称定义
  - name: redis-pv-claim
    persistentVolumeClaim:
      claimName: redis-pv-claim
> kubectl create -f volume-local.yaml
pod "volume-local-pod" created
```

![](Kubernetes(%E4%B8%83)%20-%20Volume/5032C9A5-07CE-4126-9492-5B7FF6D97031.png)

这个时候容器的节点在K8S-S1上我们看一下是否保存到了K8S-S1的磁盘上了吗

![](Kubernetes(%E4%B8%83)%20-%20Volume/11BD6E61-A614-4D62-ABCE-4795D9EAE5A1.png)


## 2.NAS网络盘
但是这样做有一个很大的弊端,如果这个Pod重启可能会被调度到其他的节点上,那么对应挂载盘的就会情况,这里有两种方式解决,第一种就是固定Pod运行的节点,在就是使用共享磁盘(首先你需要创建一个NAS盘)

一般用的比较频繁的就是NAS盘作为挂载盘,用法如下

```
> vim nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: xxxxxx.cn-hangzhou.nas.aliyuncs.com   # nfs的地址
    path: "/"                                     # nfs的挂载目录(一定需要有这个文件目录)
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pv
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 5Gi

> kubectl create -f nfs-pv.yaml
persistentvolume "nfs-pv" created
persistentvolumeclaim "nfs-pv" created
```

我们创建两个Pod共享一个NAS盘

```
> vim volume-nfs.yaml
apiVersion: extensions/v1beta1 
kind: Deployment
metadata:
  name: volume-nfs
spec:
  replicas: 2  
  template:
    metadata:
      labels:                                   # 容器的标签 可和service关联
        app: volume-nfs
    spec:
      containers:
      - name: mypod
        image: redis
        volumeMounts:                         # 磁盘挂载
          - name: nfs-pv
            mountPath: "/etc/redis"
      volumes:                                  # 磁盘挂载别称定义
      - name: nfs-pv
        persistentVolumeClaim:
          claimName: php-general-test
> kubectl create -f volume-nfs.yaml
deployment "volume-nfs" created
```

两个Pod分别在不同的节点中

![](Kubernetes(%E4%B8%83)%20-%20Volume/9900E8A3-3DA4-4E2D-9E61-B97B0AD769CC.png)

![](Kubernetes(%E4%B8%83)%20-%20Volume/2FF53CE0-882A-43B2-93EE-3CDC72A2052A.png)

![](Kubernetes(%E4%B8%83)%20-%20Volume/3D4BC685-4112-4676-8EDB-F001DB905F93.png)


## 3. 其他Volume支持类型

具体使用明细可以参考官方文档: [Volumes | Kubernetes](https://v1-8.docs.kubernetes.io/docs/concepts/storage/volumes/)

- awsElasticBlockStore
- azureDisk
- azureFile
- cephfs
- downwardAPI
- emptyDir
- fc （光纤通道）
- flocker
- gcePersistentDisk
- gitRepo
- glusterfs
- hostPath
- iscsi
- local
- nfs
- persistentVolumeClaim
- projected
- portworxVolume
- quobyte
- rbd
- scaleIO
- secret
- storageos
- vsphereVolume

