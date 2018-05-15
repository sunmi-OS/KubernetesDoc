# Kubernetes(十) - CI/CD自动更新和Go客户端


![](Kubernetes(%E5%8D%81)%20-%20CICD%E8%87%AA%E5%8A%A8%E6%9B%B4%E6%96%B0%E5%92%8CGo%E5%AE%A2%E6%88%B7%E7%AB%AF/1B88873A-A973-4B22-A7BB-945B4E30394E.png)



Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  


默认是不会按照TAG号更新就算删除也不会更新
imagePullPolicy几种方式
 imagePullPolicy: Always
