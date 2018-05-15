# Kubernetes(二) - 使用Rancher部署K8S集群(搭建Rancher)

![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/10.png)

众所周知Kubernetres虽然很好但是安装部署很复杂,
Rancher功能很强大,我们这里仅仅使用Rancher来搭建管理Kubernetes集群

Kubernetes官方文档:[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)
Kubernetes官方Git地址:[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)
Rancher官方地址: [https://www.cnrancher.com/](https://www.cnrancher.com/)  

> PS:本系列中使用 KubernetesV1.8 RancherV1.6.14  

## 1.安装Rancher

这里使用三台机器来搭建Kubernetes集群
- K8S-M  172.168.0.128
- K8S-S1 172.168.0.129
- K8S-S2 172.168.0.130

Rancher Server当前版本中有2个不同的标签。对于每一个主要的release标签，我们都会提供对应版本的文档。
- rancher/server:latest 此标签是最新一次开发的构建版本。这些构建已经被CI框架自动验证测试。但这些release并不代表可以在生产环境部署。
- rancher/server:stable 此标签最新一个稳定的release构建。这个标签代表推荐在生产环境中使用的版本。

> PS:请不要使用任何带有 rc{n} 前缀的release。这些构建都是Rancher团队的测试构建。  

这里使用Cenos7.4,并且安装好Docker-17.03.2-ce版本,在拉取稳定的Rancher-v1.6.14版本

> PS:Kubernets支持的Docker版本 1.11.2 to 1.13.1 and 17.03.2

```bash
docker pull rancher/server:v1.6.14
```

使用一个简单的命令就可以启动一个单实例的Rancher。
```bash
> docker run -d --restart=unless-stopped -p 8080:8080 rancher/server:v1.6.14
```

关闭防火墙(后续增加节点需要和主节点端口通讯需要关闭防火墙)
```bash
> systemctl stop firewalld.service    # 关闭firewall
> systemctl disable firewalld.service # 禁止firewall开机启动
```

等待容器启动访问对应IP的8080端口的地址可以看到如下界面

![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/11.png)

通过右下角可以编辑语言切换成简体中文
![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/12.png)


## 2 外挂数据库目录(按需)

在Rancher Server容器中，如果你想使用一个主机上的卷来持久化数据库，如下命令可以在启动Rancher时挂载MySQL的数据卷。
```bash
> docker run -d -v /usr/local/rancher_mysql:/var/lib/mysql --restart=unless-stopped -p 8080:8080 rancher/server:stable
```

使用这条命令，数据库就会持久化在主机上。如果你有一个现有的Rancher Server容器并且想挂在MySQL的数据卷，可以参考官方的Rancher升级介绍。

### Rancher使用外部数据库
除了使用内部的数据库，你可以启动一个Rancher Server并使用一个外部的数据库。启动命令与之前一样，但添加了一些额外的参数去说明如何连接你的外部数据库。

> 注意：在你的外部数据库中，只需要提前创建数据库名和数据库用户。Rancher会自动创建Rancher所需要的数据库表。  

以下是创建数据库和数据库用户的SQL命令例子
```bash
> CREATE DATABASE IF NOT EXISTS cattle COLLATE = 'utf8_general_ci' CHARACTER SET = 'utf8';
> GRANT ALL ON cattle.* TO 'cattle'@'%' IDENTIFIED BY 'cattle';
> GRANT ALL ON cattle.* TO 'cattle'@'localhost' IDENTIFIED BY 'cattle';
```

启动一个Rancher连接一个外部数据库，你需要在启动容器的命令中添加额外参数。

```bash
docker run -d --restart=unless-stopped -p 8080:8080 rancher/server \
    --db-host myhost.example.com --db-port 3306 --db-user username --db-pass password --db-name cattle
```

## 3 权限管理

机制的小伙伴都注意到了现在登录到Rancher不需要任何用户名密码,Rancher的用户体系需要自己开启

![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/13.png)

可以选择很多汇总认证的方式

![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/14.png)

最方便的方式就是开启本地账号认证

![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/15.png)

填写好相关用户名密码之后开启本地验证下次登录就需要验证用户了,并且在后续的管理中也能进行权限控制

![](https://github.com/sunmi-OS/KubernetesDoc/blob/master/src/images/16.png)


## 4 Rancher多节点HA部署

在高可用(HA)的模式下运行Rancher Server与使用外部数据库运行Rancher Server一样简单，需要暴露一个额外的端口，添加额外的参数到启动命令中，并且运行一个外部的负载均衡就可以了。

HA部署需求

- HA 节点:
	- 所有安装有支持的Docker版本的现代Linux发行版 RancherOS, Ubuntu, RHEL/CentOS 7 都是经过严格的测试。
		- 对于 RHEL/CentOS, 默认的 storage driver, 例如 devicemapper using loopback, 并不被Docker推荐。 请参考Docker的文档去修改使用其他的storage driver。
		- 对于 RHEL/CentOS, 如果你想使用 SELinux, 你需要 安装额外的 SELinux 组件.
	- 9345, 8080 端口需要在各个节点之间能够互相访问
	- 1GB内存
- MySQL数据库
	- 至少 1 GB内存
	- 每个Rancher Server节点需要50个连接 (例如：3个节点的Rancher则需要至少150个连接)
	- MYSQL配置要求
		- 选项1: 用默认COMPACT选项运行Antelope
		- 选项2: 运行MySQL 5.7，使用Barracuda。默认选项ROW_FORMAT需设置成Dynamic
- 外部负载均衡服务器
	- 负载均衡服务器需要能访问Rancher Server节点的 8080 端口

> 注意：目前Rancher中并不支持Docker for Mac  

## 5.大规模部署建议

- 每一个Rancher Server节点需要有4 GB 或者8 GB的堆空间，意味着需要8 GB或者16 GB内存
- MySQL数据库需要有高性能磁盘
- 对于一个完整的HA，建议使用一个有副本的Mysql数据库。另一种选择则是使用Galera集群并强制写入一个MySQL节点。

在每个需要加入Rancher Server HA集群的节点上，运行以下命令：
```bash
# Launch on each node in your HA cluster
> docker run -d --restart=unless-stopped -p 8080:8080 -p 9345:9345 rancher/server \
     --db-host myhost.example.com --db-port 3306 --db-user username --db-pass password --db-name cattle \
     --advertise-address <IP_of_the_Node>
```

在每个节点上，<IP_of_the_Node> 需要在每个节点上唯一，因为这个IP会被添加到HA的设置中。
如果你修改了 -p 8080:8080 并在host上暴露了一个不一样的端口，你需要添加 --advertise-http-port <host_port> 参数到命令中。

> 注意：你可以使用 docker run rancher/server --help 获得命令的帮助信息  

### HA模式下的RANCHER SERVER节点
如果你的Rancher Server节点上的IP修改了，你的节点将不再存在于Rancher HA集群中。你必须停止在--advertise-address配置了不正确IP的Rancher Server容器并启动一个使用正确IP地址的Rancher Server的容器。


