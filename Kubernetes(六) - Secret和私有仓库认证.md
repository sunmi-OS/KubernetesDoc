# Kubernetes(六) - Secret和私有仓库认证
#w-blog博客/kube

![](Kubernetes(%E5%85%AD)%20-%20Secret%E5%92%8C%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93%E8%AE%A4%E8%AF%81/1B88873A-A973-4B22-A7BB-945B4E30394E.png)

对一个公司来说安全也是最为重要的因为可能一旦出现安全问题可能这个公司就完了,所以对密码管理是一个长久不变的话题,Kubernetes对密码管理提供了Secret组件进行管理,最终映射成环境变量,文件等方式提供使用,统一进行了管理更换方便,并且开发人员并不需要关心密码降低了密码的受众范围从而保障了安全.

附上:

喵了个咪的博客:[w-blog.cn](w-blog.cn)
Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  

## 1. 初始化Secret
首先我们需要初始化一个Secret,使用Yaml文件创建时需要使用base64之后的内容作为Value
```
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

老规矩通过yaml的方式创建我们的Secret配置文件可以看到已经生效了

```
> vim secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm

> kubectl create -f ./secret.yaml
secret "mysecret" created
> kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-lnftf   kubernetes.io/service-account-token   3         1d
mysecret              Opaque                                2         9s
```



## 2. 环境变量
我们在使用Secret第一个场景就是作为容器的环境变量,大部分容器都提供使用环境变量配置密码的功能,你的程序只需要读取到这个环境变量使用这个环境变量的内容去链接到对应的服务就可以正常使用了,如下我们初始化一个Pod服务,使用之前预设好的信息作为用户名密码配置进去

```
> vim secret-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
> kubectl create -f secret-env.yaml
```

![](Kubernetes(%E5%85%AD)%20-%20Secret%E5%92%8C%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93%E8%AE%A4%E8%AF%81/603DFD84-51E4-4A5E-8DB9-7899D8C47C4D.png)


## 3.文件(TLS证书)

除了配置成环境变量我们在很多地方也会使用到文件的方式来存放密钥信息,最常用的就是HTTPS这样的TLS证书,使用证书程序(比如Nginx没法使用环境变量来配置证书)需要一个固定的物理地址去加载这个证书,我们吧之前配置用户名和密码作为文件的方式挂在到某个目录下

```
> vim secret-file.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-file-pod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
> kubectl create -f secret-file.yaml
```

![](Kubernetes(%E5%85%AD)%20-%20Secret%E5%92%8C%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93%E8%AE%A4%E8%AF%81/827F06D0-3613-4EBD-8C4A-85038985E610.png)


如果有需要对同的配置分开挂载到不同的地方可以使用如下配置

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-file-pod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

- username存储在/etc/foo/my-group/my-username文件而不是/etc/foo/username。
- password 不会挂载到磁盘

因为映射成了文件那么对权限也是可以控制的

```
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 256
```

然后，秘密将被挂载，/etc/foo并且由秘密卷挂载创建的所有文件都将具有权限0400。
> PS: JSON规范不支持八进制表示法，因此对于0400权限使用值256。如果您使用yaml代替pod的json，则可以使用八进制表示法以更自然的方式指定权限。  
您也可以使用映射（如上例所示），并为不同的文件指定不同的权限，如下所示：

```
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
        mode: 511
```

在这种情况下，生成的文件/etc/foo/my-group/my-username将具有权限值0777。由于JSON限制，您必须以十进制表示法指定模式。

## 4.Docker私有仓库认证
使用过K8s的小伙伴肯定会遇到一个问题,我们在使用自有的Docker仓库的时候都需要先登录用户名和密码,但是如果使用K8S怎么配置密码呢?在secret中有一个类型是docker-registry我们可以通过命令行的方式创建在获取Docker镜像时使用的用户名和密码

```
kubectl create secret docker-registry  regsecret --docker-server=registry-vpc.cn-hangzhou.aliyuncs.com --docker-username=admin --docker-password=123456 --docker-email=xxxx@qq.com
```

如果使用编排文件是如下格式
```
kind: Secret
apiVersion: v1
metadata:
  name: regsecret
type: kubernetes.io/dockercfg
data:
  ".dockercfg": eyJyZWdpc3RyeS12cGMuY24taGFuZ3pob3UuYWxpeXVuY3MuY29tIjp7InVzZXJuYW1lIjoiYWRtaW4iLCJwYXNzd29yZCI6IjEyMzQ1NiIsImVtYWlsIjoieHh4eEBxcS5jb20iLCJhdXRoIjoiWVdSdGFXNDZNVEl6TkRVMiJ9fQ==

# 反base64的结果 : {"registry-vpc.cn-hangzhou.aliyuncs.com":{"username":"admin","password":"123456","email":"xxxx@qq.com","auth":"YWRtaW46MTIzNDU2"}}
```

然后我们就可以在获取指定镜像的时候为他指定一个获取镜像的Docker凭证

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-file-pod
spec:
  containers:
  - name: mypod
    image: redis
  imagePullSecrets:                         # 获取镜像需要的用户名密码
   - name: regsecret
```


