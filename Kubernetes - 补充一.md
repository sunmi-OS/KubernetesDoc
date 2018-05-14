# Kubernetes - 补充一
#w-blog博客/kube

![](Kubernetes%20-%20%E8%A1%A5%E5%85%85%E4%B8%80/1B88873A-A973-4B22-A7BB-945B4E30394E.png)



附上:

喵了个咪的博客:[w-blog.cn](w-blog.cn)
Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  


## Kubectl命令

## 对整个namespce进行资源限制

## 权限验证

## 弹性扩容实践


## NodePort方式暴露服务的端口的默认范围（30000-32767）修改
则在apiserver的启动命令里面添加如下参数 –service-node-port-range=1-65535
