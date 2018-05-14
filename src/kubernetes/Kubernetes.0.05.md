# Kubernetes(五) - Service


Kubernetes解决的另外一个痛点就是服务发现,服务发现机制和容器开放访问都是通过Service来实现的,把Deployment和Service关联起来只需要Label标签相同就可以关联起来形成负载均衡,基于kuberneres的DNS服务我们只需要访问Service的名字就能以负载的方式访问到各个容器

Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)

Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  

## 1. Service的三种类型
Service有三种类型：

- ClusterIP：默认类型，自动分配一个仅cluster内部可以访问的虚拟IP
常用于内部程序互相的访问,比如Gitlab需要访问Redis的postgresql,但是是内部使用的不需要外部访问,这个时候用ClusterIP就比较合适

- NodePort：在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过<NodeIP>:NodePort来访问改服务
当我们的Gitlab需要提供访问,可以使用NodePort指定一个端口释放服务,然后外层负载均衡映射就可以在外部访问,或者直接访问对应的端口

> PS:NodePort方式暴露服务的端口的默认范围（30000-32767）如果需要修改则在apiserver的启动命令里面添加如下参数 –service-node-port-range=1-65535  

- LoadBalancer：在NodePort的基础上，借助cloud provider创建一个外部的负载均衡器，并将请求转发到<NodeIP>:NodePort
LoadBalancer是NodePort的升级版本,相当于和cloud provider结合不需要手动指定

我们经常使用的还是上面前两种方式,我们先创建一个nginx-deployment镜像2个Pod以便于接下来的使用

```
> vim nginx-deployment.yaml

apiVersion: extensions/v1beta1                  # K8S对应的API版本
kind: Deployment                                # 对应的类型
metadata:
  name: nginx-deployment
  labels:
    name: nginx-deployment
spec:
  replicas: 2                                   # 镜像副本数量
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
```

![](Kubernetes(%E4%BA%94)%20-%20Service/50.png)

我们分别修改一下对应的输出
```
> echo nginx1 > /usr/share/nginx/html/index.html
> echo nginx2 > /usr/share/nginx/html/index.html
```

安装好ping 和 curl
```
> apt-get update
> apt-get install curl iputils-ping
```


## 2. ClusterIP
```
> vim test-clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: test-clusterip-service             # 名称
  labels:
    name: test-clusterip-service
spec:
  type: ClusterIP                               # 开发端口的类型
  selector:                                     # service负载的容器需要有同样的labels
    app: nginx
  ports:
  - port: 80                                    # 通过service来访问的端口
    targetPort: 80                              # 对应容器的端口
> kubectl create -f test-clusterip-service.yaml
service "test-clusterip-service" created
> kubectl get service
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes               ClusterIP   10.43.0.1      <none>        443/TCP   12d
test-clusterip-service   ClusterIP   10.43.202.97   <none>        80/TCP    18s
```

应为selector关联上了Deployment的Label所以直接访问是可以访问到的具体nginx镜像内,而且应为是两个Pod所以是负载均衡之前我们修改了输出多次访问可以看到不一样的结果,基于Kube-DNS也可以使用Service名称进行访问

```
root@nginx-deployment-68fcbc9696-wbgxr:/usr/share/nginx/html# curl 10.43.202.97
nginx2
root@nginx-deployment-68fcbc9696-wbgxr:/usr/share/nginx/html# curl test-clusterip-service
nginx1
```

## 3. NodePort

NodePort设计出来的主要目的就是对外部放出服务,也就是被外部能够访问到,

```
> vim test-nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: test-nodeport-service             # 名称
  labels:
    name: test-nodeport-service
spec:
  type: NodePort                               # 开发端口的类型
  selector:                                     # service负载的容器需要有同样的labels
    app: nginx
  ports:
  - port: 80                                    # 通过service来访问的端口
    targetPort: 80                              # 对应容器的端口
    nodePort: 30080                             # 对应需要放到宿主机IP上的端口

> kubectl create -f test-nodeport-service.yaml
service "test-nodeport-service" created
> kubectl get service
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes               ClusterIP   10.43.0.1      <none>        443/TCP        12d
test-clusterip-service   ClusterIP   10.43.202.97   <none>        80/TCP         14m
test-nodeport-service    NodePort    10.43.101.60   <none>        80:30080/TCP   5s
```

此时在宿主机上的30080端口就可以访问到我们的两个Nginx容器了,如果机器绑定的有IP的话就可以直接访问或者在使用负载均衡对外放出服务

> 只有在KUbernetes中才会受到Kube-DNS的影响,在宿主机上无法使用test-nodeport-service访问Service只能通过NodePort进行访问  

```
[root@k8s-m ~]# curl 127.0.0.1:30080
nginx2
[root@k8s-m ~]# curl 127.0.0.1:30080
nginx1
```

## 4. Ingress

Service主要是处理4层TCP负载,但是往往对外需要放出HTTP七层协议的服务,一般我们在一套集群下如果有多个HTTP服务会使用Nginx来统一接受80端口的数据然后通过域名或者是访问路径来选择不同的服务,Ingress就是解决这个问题诞生的,Ingress可以和Service结合对80端口的访问更具域名的过滤和访问路径的过滤路由到对应的service,我们一起看几个例子:

```
foo.bar.com -> 172.168.0.128 -> / foo    nginx:80
                                / bar    nginx:80
```

```
> vim ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: nginx
          servicePort: 80
      - path: /bar
        backend:
          serviceName: nginx
          servicePort: 80

> kubectl create -f ing.yaml
> kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -
          foo.bar.com
          /foo          nginx:80
          /bar          nginx:80
```


```
foo.bar.com --|                 |-> foo.bar.com nginx:80
              | 172.168.0.128   |
bar.foo.com --|                 |-> bar.foo.com nginx:80
```

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```

支持TLS需要使用Secret预先配置好对应的证书
```
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: testsecret
  namespace: default
type: Opaque
```

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: no-rules-map
spec:
  tls:
  - secretName: testsecret
  backend:
    serviceName: nginx
    servicePort: 80
```

> 使用Ingress能实现部分功能,但是笔者推荐使用网关服务(比如kong)来进行处理会更具灵活功能更下强大可控  

## 小技巧

- 跨namespace访问Service
到这里我们还没有展开说NameSpace,NameSpace隔离了资源,比如你在A中不能创建两个名字一样的Service(Kubernetes其他资源同理),但是创建出一个NameSpace的是可以创建名字为Nginx的Service的
这个时候你在A空间中访问nginx-service的是A空间的容器,在B空间访问是B空间的nginx-service
但是如果需要在B空间访问A空间的Nginx-service要怎么办呢?

Kube提供在访问Service的时候末尾增加NameSpace名字进行访问,在B空间访问A的nginx-service可以使用nginx-service.A 进行访问
