# Kubernetes(四) - Pod和Deployment

![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/1B88873A-A973-4B22-A7BB-945B4E30394E.png)

Kubernetes中有各种各样的组件,对于容器来说Kubernetes最小的单元是由Pod进行组成的,但是我们在使用过程中经常会使用到Deployment来部署我们的应用,其中究竟区别在哪里,我们今天就来一同探索


Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  

## 1.Pod最小的单元

Pod封装了一个或多个应用程序的容器(比如nginx等),存储资源,唯一的网络IP以及管理容器的一些选项
Pod标示的是一个部署单元,可以理解为Kubernetes中的应用程序的单个实例,它可能由单个容器组成,也可能由少量紧密耦合并共享资源的容器组成。

> 如果多个容器在同一Pod下他们公用一个IP所以不能出现重复的端口号,比如在一个Pod下运行两个nginx就会有一个容器异常,一个Pod下的多个容器可以使用localhost来访问对方端口  

> 应为Pod是最小的单元如果在Pod中容器出现异常终止了是不会重启,在实际使用场景下基本不会直接使用Pod而是使用Deployment部署自己的应用  

例子:

```
> vim myapp-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']

> kubectl create -f myapp-pod.yaml
> kubectl get pod
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          8s
```

可以在web页面中查看到具体的pod(UI上称为容器组)
![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/44.png)
查看日志可以看到
![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/45.png)
删除Pod
```
> kubectl delete -f myapp-pod.yaml
pod "myapp-pod" deleted
```

向上述所说的我们也可以在一个Pod下运行多个容器(注意不要端口冲突)

```yaml
> vim nginx-mysql-pod.yaml
apiVersion: v1                          # api版本
kind: Pod                               # 组件类型
metadata:
  name: nginx-mysql-pod
  labels:                               # 标签
    app: nginx-mysql
spec:
  containers:
  - name: nginx                         # 名称
    image: nginx                        # image地址
  - name: mysql                         
    image: mysql                        
    env:                                # 环境变量
    - name: MYSQL_ROOT_PASSWORD
      value: mysql

> kubectl create -f nginx-mysql-pod.yaml
> kubectl get pod
NAME              READY     STATUS    RESTARTS   AGE
nginx-mysql-pod   2/2       Running   0          1m
```

在ui中就可以看到一个Pod下运行着两个容器

![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/46.png)

通过运行命令可以进入到那个容器的终端,这里选择mysql容器

![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/47.png)

这里系统没有curl这里安装好了curl访问本地80端口,能访问到nginx容器的内容(这里证明了在一个Pod下的网络是共享的)

![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/48.png)

## 2.Deployment部署

在早期版本使用Replication Controller对Pod副本数量进行管理,在新的版本中官方推荐使用Deployment来代替RC,Deployment相对RC有这些好处

- Deployment拥有更加灵活强大的升级、回滚功能,并且支持滚动更新
- 使用Deployment升级Pod只需要定义Pod的最终状态，k8s会为你执行必要的操作(RC要自己定义如何操作)

不管是RC还是Deployment解决的主要问题是,每个Pod都运行给定应用程序的单个实例。如果您想水平扩展应用程序（例如，运行多个同样的实例），则应该使用多个Pod。这里带来的Pod管理成本

```yaml
> vim nginx-deployment.yaml

apiVersion: extensions/v1beta1                  # K8S对应的API版本
kind: Deployment                                # 对应的类型
metadata:
  name: nginx-deployment
  labels:
    name: nginx-deployment
spec:
  replicas: 1                                   # 镜像副本数量
  template:
    metadata:
      labels:                                   # 容器的标签 可和service关联
        app: nginx
    spec:
      containers:
        - name: nginx                          # 容器名和镜像
          image: nginx
          imagePullPolicy: Always

> kubectl create -f nginx-deployment.yaml
> kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-68fcbc9696-tr5fm   1/1       Running   0          6s
nginx-mysql-pod                     2/2       Running   0          15m
> kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1         1         1            1           2m
```

因为使用deployment会部署很多个Pod所以Pod的名字后面会带一串随机数避免重复

![](Kubernetes(%E5%9B%9B)%20-%20Pod%E5%92%8CDeployment/49.png)

扩容
```
> kubectl scale deployment nginx-deployment --replicas=3
> kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-68fcbc9696-9r4p8   1/1       Running   0          22s
nginx-deployment-68fcbc9696-lkff2   1/1       Running   0          22s
nginx-deployment-68fcbc9696-tr5fm   1/1       Running   0          5m
nginx-mysql-pod                     2/2       Running   0          21m
```

恢复一个

```
> kubectl scale deployment nginx-deployment --replicas=1
> kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-68fcbc9696-tr5fm   1/1       Running   0          6m
nginx-mysql-pod                     2/2       Running   0          21m
```

最关键的功能就是可以弹性扩容根据CPU的占用率(需要结合资源限制一同使用),后面会进行实际演示

```
> kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=80
```

Deployment回保障你的容器的运行状态,如果删除Pod,Deployment会立即重启一个
```
> kubectl delete pod nginx-deployment-68fcbc9696-tr5fm
pod "nginx-deployment-68fcbc9696-tr5fm" deleted

> kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-68fcbc9696-ndfzj   1/1       Running   0          12s
nginx-mysql-pod                     2/2       Running   0          32m
```

> Deployment会配合RC调度启动新的Pod从而保障定量的Pod数量  

## 3.环境变量配置和执行命令参数配置

### 3.1 环境变量配置

环境变量在上述使用多Pod的时候已经提到了,但是这里面**有一个坑就是Value值不能数字开头**,否则会报错无法创建,需要使用引号应用起来

```yaml
> vim mysql-pod.yaml
apiVersion: v1                          # api版本
kind: Pod                               # 组件类型
metadata:
  name: mysql-pod
  labels:                               # 标签
    app: mysql
spec:
  containers:
  - name: mysql                         # 名称
    image: mysql                        # image地址
    env:                                # 环境变量
    - name: MYSQL_ROOT_PASSWORD
      value: "666666"

> kubectl create -f mysql-pod.yaml
Error from server (BadRequest): error when creating "mysql-pod.yaml": Pod in version "v1" cannot be handled as a Pod: v1.Pod: Spec: v1.PodSpec: Containers: []v1.Container: v1.Container: Env: []v1.EnvVar: v1.EnvVar: Value: ReadString: expects " or n,parsing 180 ...,"value":6... at {"apiVersion":"v1","kind":"Pod","metadata":{"labels":{"app":"mysql"},"name":"mysql-pod","namespace":"default"},"spec":{"containers":[{"env":[{"name":"MYSQL_ROOT_PASSWORD","value":666666}],"image":"mysql","name":"mysql"}]}}

# 修改
    env:                                # 环境变量
    - name: MYSQL_ROOT_PASSWORD
      value: "666666"

> kubectl create -f mysql-pod.yaml
pod "mysql-pod" created

> kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
mysql-pod                           1/1       Running   0          12s
nginx-deployment-68fcbc9696-ndfzj   1/1       Running   0          10m
nginx-mysql-pod                     2/2       Running   0          42m
```

### 3.2 执行命令参数

我们在使用Docker的时候为了运行具体的程序一般会在Dockerfile中使用CMD预设好需要执行的命令,当然也可以在运行的时候替换为你希望执行的命名,比如之前我们运行的输出程序就使用了command执行我们希望的命令

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']

```

但是除了CMD的方式有很多复杂组件的Docker使用的是**ENTRYPOINT**的方式(使用sh文件接收参数运行复杂程序),这个使用又有一个坑,如果使用command配置参数会出现设置的参数无效(docker-composer是可以使用cmd来传参的),在这里需要使用args的方式指定参数,举个例子(以下例子无法正常运行只是展示):

```
apiVersion: extensions/v1beta1                  # K8S对应的API版本
kind: Deployment                                # 对应的类型
metadata:
  name: kong-deployment
  labels:
    name: kong-deployment
spec:
  replicas: 1                                   # 镜像副本数量
  template:
    metadata:
      labels:                                   # 容器的标签 可和service关联
        app: kong
    spec:
      containers:
        - name: kong                            # 容器名和镜像
          image: kong:0.11.2
          imagePullPolicy: Always
          args: ["kong","migrations","up"]      # 执行数据库初始化
```
