# Kubernetes(一) - Docker管理工具
#w-blog博客/kube

![](Kubernetes(%E4%B8%80)%20-%20Docker%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7/1B88873A-A973-4B22-A7BB-945B4E30394E.png)

虽然Docker已经很强大了,但是在实际使用上还是有诸多不便,比如集群管理,资源调度文件管理等等,那么在这样一个百花齐放的容器时代涌现出了很多解决方案,比如Swarm,Mesos,Kubernetes等等,其中谷歌开源的Kubernetes是作为老大哥的存在,从本节开始将介绍如何打造自己的Kubernetes,并且了解它各个组件的用途

附上:

喵了个咪的博客:[w-blog.cn](w-blog.cn)
Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)

## 1. 为什么选择kubernetes容器管理 
      
Docker宣布在下一个企业版本开始支持Kubernetes。然而在Dockercon Europe 2017之前，Kuberenetes是人们避而不谈的大象。
三个主要云提供商都加入了由Kubernetes主持的开源基金会，即云原生基金会（CNCF），这让其势不可挡。

Docker让容器变成了主流。自从项目发布以来，Docker着重于提升开发者的体验。基本理念是可以在整个行业中，
在一个标准的框架上，构建、交付并且运行应用。通常来讲，我们部署项目会构建出一个持续集成和持续开发的流程，然后将其应用到生产环境。

作为编排工具，从社区的年龄来讲，Kubernetes不占优势。毕竟Kubernetes才两岁而已（从作为开源项目算起），而Apache的Mesos已经推出7年之久。Docker Swarm虽然是比Kubernetes更年轻的项目，但是它的背后是来自于Docker官方容器中心的全方位支持。

然由于Kubernetes社区在基础云平台的管理下正在不断变得丰富多彩。

* Kubernetes是活跃在Github中前几名的项目之一：占有在所有项目中排名0.01%的star，而且在所有团队项目活跃度排名第一。

* 虽然Kubernetes的文档欠佳，但是Kubernetes有自己的Slack和Stack Overflow社区作为补充，帮助解决问题优于其竞争对手。

* 在LinkedIn上有更专业的Kubernetes专家，相比其他工具，Kubernetes通过LinkedIn为使用者提供了更广阔的解决问题空间。

* 通过OpenHub的数据却显示了Apache Mesos正在走向衰落，Docker Swarm增长也开始放缓。从原始社区的贡献来讲，Kubernetes正在迅速增长，从1000+贡献者34000+的提交贡献，远远超过了其他像Mesos竞争对手的四倍之多。

这样的原因肯定是因为谷歌，或者是说谷歌的选择开源。虽然其他的每一个编排项目背后都有一个供应商公司在影响着，但是Kubernetes受益于谷歌的不干涉开发，以及比较优秀的原始引擎。

与此同时，Docker拥有实际上的容器标准，Docker也一直在努力构建与Kubernetes一样广泛深入的容器社区。基于以上原因，谷歌的Kelsey Hightower指出，Docker本身在阻止竞争对手进入构建容器标准。


相对于原生容器来说,在本地跑跑环境还行但是不熟到生产环境可是要打一个问号,稳定性?健壮性?可监控?部署?这些都是需要解决问题,以下三点其实就包括了为什么需要Kubernetes容器管理:

- Kubernetes是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

- Kubernetes一个核心的特点就是能够自主的管理容器来保证云平台中的容器按照用户的期望状态运行着（比如用户想让apache一直运行，用户不需要关心怎么去做，Kubernetes会自动去监控，然后去重启，新建，总之，让apache一直提供服务），管理员可以加载一个微型服务，让规划器来找到合适的位置，同时，Kubernetes也系统提升工具以及人性化方面，让用户能够方便的部署自己的应用（就像canary deployments）。

- Kubernetes是为生产环境而设计的容器调度管理系统，对于**负载均衡、服务发现、高可用、滚动升级、自动伸缩**等容器云平台的功能要求有原生支持。



## 2.Kubernetes解决的核心问题

- 负载均衡 - 多个同样的容器运行着多套,Kube-Service提供了统一的访问定义,以负载均衡的方式来提供访问
- 服务发现 - Kube-Service和Kube-DNS结合,只需要通过固定的Kube-Service名称就可以访问到对应的容器,不需要独立寻找使用服务发现组件
- 高可用  - Kube会检查服务的健康状态,会不停尝试重新启动服务,保障正常运行
- 滚动升级 - 在升级过程中Kube会有规划的挨个容器滚动升级,把升级带来的影响降低到最小
- 自动伸缩 - 可以配置策略当容器资源使用较高会自动增加一个新的容器来分担压力,当资源使用率降低会回收容器
- 快速部署 - 使用Kube编写好对应的编排问题,可以在及短的时间部署一套环境
- 资源限制 - 对程序限制最大资源使用量避免抢占资源遇到事故或压力也能从容保障基础服务不受影响

下图是Kube内部组件协助运行图:

![](Kubernetes(%E4%B8%80)%20-%20Docker%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7/669282A1-83CB-458B-94AB-32D3F668273B.png)


## 3.Kubernetes组件和核心技术概念

一个K8s集群是由分布式存储（etcd）、服务节点（Minion，etcd现在称为Node）和控制节点（Master）构成的。所有的集群状态都保存在etcd中，Master节点上则运行集群的管理控制模块。Node节点是真正运行应用容器的主机节点，在每个Minion节点上都会运行一个Kubelet代理，控制该节点上的容器、镜像和存储卷等。



Kubernetes主要由以下几个核心组件组成：

* etcd保存了整个集群的状态；

* apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；

* controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；

* scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；

* kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；

* Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；


此外Kubernetes中使用的各个组件的主要概念：

1. Cluster : 集群是指由Kubernetes使用一系列的物理机、虚拟机和其他基础资源来运行你的应用程序。
2. Node : 一个node就是一个运行着Kubernetes的物理机或虚拟机，并且pod可以在其上面被调度.
3. Pod : 一个pod对应一个由相关容器和卷组成的容器组.
4. Label : 一个label是一个被附加到资源上的键/值对，譬如附加到一个Pod上，为它传递一个用户自定的并且可识别的属性.Label还可以被应用来组织和选择子网中的资源.
5. selector是一个通过匹配labels来定义资源之间关系得表达式，例如为一个负载均衡的service指定所目标Pod.
6. Replication Controller : replication controller 是为了保证一定数量被指定的Pod的复制品在任何时间都能正常工作.它不仅允许复制的系统易于扩展，还会处理当pod在机器在重启或发生故障的时候再次创建一个.
7. Service : 一个service定义了访问pod的方式，就像单个固定的IP地址和与其相对应的DNS名之间的关系。
8. Volume: 一个volume是一个目录，可能会被容器作为未见系统的一部分来访问。Kubernetes volume 构建在Docker Volumes之上,并且支持添加和配置volume目录或者其他存储设备.
9. Secret : Secret 存储了敏感数据，例如能允许容器接收请求的权限令牌。
10. Name : 用户为Kubernetes中资源定义的名字.
11. Namespace : Namespace 好比一个资源名字的前缀。它帮助不同的项目、团队或是客户可以共享cluster,例如防止相互独立的团队间出现命名冲突.
12. Annotation : 相对于label来说可以容纳更大的键值对，它对我们来说可能是不可读的数据，只是为了存储不可识别的辅助数据，尤其是一些被工具或系统扩展用来操作的数据.


Kubernetes设计理念和功能其实就是一个类似Linux的分层架构：

* 核心层：Kubernetes最核心的功能，对外提供API构建高层的应用，对内提供插件式应用执行环境

* 应用层：部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS解析等）

* 管理层：系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、动态Provision等）以及策略管理（RBAC、Quota、PSP、NetworkPolicy等）

* 接口层：kubectl命令行工具、客户端SDK以及集群联邦

* 生态系统：在接口层之上的庞大容器集群管理调度的生态系统，可以划分为两个范畴
  * Kubernetes外部：日志、监控、配置管理、CI、CD、Workflow、FaaS、OTS应用、ChatOps等
  * Kubernetes内部：CRI、CNI、CVI、镜像仓库、Cloud Provider、集群自身的配置和管理等


![](Kubernetes(%E4%B8%80)%20-%20Docker%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7/D5944419-DEA3-4967-ACCE-EFC834A2D441.png)

注:笔者能力有限有说的不对的地方希望大家能够指出,也希望多多交流!


