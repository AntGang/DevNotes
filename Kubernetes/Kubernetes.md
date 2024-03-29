# kubernetes

kubernetes是一个**软件系统**，依赖于Linux容器的特性来**运行异构的应用**，它将底层基础设施抽象，简化应用的开发，部署和运维

## 基础概念

- Pod
  - 资源清单
  - Pod的生命周期
- 控制器类型
  - Pod 控制器的特点以及使用定义方式
- K8S网络通讯模式

- 服务发现
  - SVC原理及其构建方式

- 服务分类

  - 有状态服务：DBMS，数据持久化存储  

  - 无状态服务：LVS，APACHE


- 存储
  - 多种存储类型的特点
  - 不同环境中选择合适的存储方案
- 调度器
  - 调度器原理
  - 根据要求把Pod定义到特定的节点运行
- 安全
  - 集群的认证，鉴权，访问控制
- HELM
  - 类似Linux的yum，HELM 原理
  - HELM自定义模板
  - HELM部署一些常用插件
- 高可用集群
  - 副本数据最好是3个以上的（3，5，7，9）

## 特点

- 开源，轻量级
- 弹性伸缩 
- 基于容器的应用部署、维护和滚动升级
- 无状态服务和有状态服务
- 负载均衡和服务发现
- 插件机制保证扩展性
- 跨机器和跨地区的集群调度

## 云计算架构

经典的云计算架构分为**三大服务层**

1. IaaS（Infrastructure as a Service）：**基础设施即服务**
1. IaaS层通过**虚拟化技术**提供计算、存储、网络等基础资源，可以在上面部署各种OS以及应用程序
   2. 开发者可以通过云厂商提供的API控制整个基础架构，无须对其进行**物理上的维护和管理**

2. PaaS（Platform as a Service）：**平台即服务**
   1. PaaS层提供软件部署平台（runtime），**抽象掉了硬件和操作系统**，可以无缝地扩展（scaling）
   2. 开发者只需要关注自己的业务逻辑，不需要关注底层

3. SaaS（Software as a Service）：**软件即服务**
   1. SaaS 层直接为开发者提供软件服务，将软件的开发、管理、部署等全部都交给第三方
   2. 用户不需要再关心技术问题，可以拿来即用  

# Kubernetes集群架构

![kubernetes架构](Kubernetes.assets/kubernetes架构.png)



## Master

集群控制节点，执行所有的命令，通常占据一个独立服务器

- Api Server（kube-apiserver）：所有服务访问唯一入口，提供HttpRest接口的服务进程
- CrontrollerManager（kube-controller-manager）：所有资源对象的自动化控制中心，维持副本期望数目
- Scheduler（kube-scheduler）：负责接受任务并实现资源调度，选择合适的节点进行分配任务
- Etcd：键值对数据库，采用http协议，储存K8S集群所有重要信息
  - 一个可信赖的分布式键值存储服务，能够为整个分布式集群存储一些关键数据，协助分布式集群的正常运转
    - 可信赖指天生支持集群化，不需要其他组件
    - 正常运转：保存分布式存储持久化的配置信息）

## Node

除了Master控制节点，Kubernetes集群中的其他工作负载节点。**Pod真正运行的主机**，可以是物理机也可以是虚拟机

为了管理Pod，每个Node节点上至少需要运行container runtime（Docker）、kubelet和kube-proxy服务

- Kubelet：直接跟容器引擎交互实现容器的生命周期管理
- Kube-proxy：负责写入规则至IpTables、Ipvs，从而实现服务映射访问，实现Kubernetes，Service的通信与负载均衡机制的重要组件
- Docker Engine（docker）：Docker引擎，负责容器创建和管理工作

> Node本质上不是Kubernetes来创建的， Kubernetes只是管理Node上的资源

## Pod

Pod是一组紧密关联的**容器集合**，是Kubernetes**调度的基本单位**

- 支持多个容器在一个Pod中**共享网络和文件系统**
- 可以通过**进程间通信**和**文件共享**这种简单高效的方式完成服务

> Pod的设计理念是每个Pod都有一个唯一的IP

### Pod特征

- 包含多个共享IPC、Network和UTC namespace的容器，可直接通过localhost通信
- **所有Pod内容器都可以访问共享的Volume，可以访问共享数据**
- Pod删除的时候先给其内的进程发送SIGTERM，等待一段时间（grace period）后才强制停止依然还在运行的进程
- **特权容器**（通过SecurityContext配置）具有改变系统配置的权限(
  - 在网络插件中大量应用
- 支持**三种重启策略**（restartPolicy）
  - Always
  - OnFailure
  - Never
- 支持**三种镜像拉取策略**（imagePullPolicy）
  - Always
  - Never
  - IfNotPresent
- Kubernetes通过**CGroup限制容器的CPU以及内存等资源**，可以设置request以及limit值
- 提供两种**健康检查探针**
  - livenessProbe：用于**探测容器是否存活**，如果探测失败，则根据重启策略进行重启操作
  - redinessProbe：用于**检查容器状态是否正常**，如果检查容器状态不正常，则请求不会到达该Pod
- **Init container在所有容器运行之前执行，常用来初始化配置**
- **容器生命周期钩子函数**，用于监听容器生命周期的特定事件，并在事件发生时执行已注册的回调函数，**支持两种钩子函数**
  - postStart：在容器启动后执行
  - preStop：在容器停止前执行

### Pod分类

**生命周期**

- **自主式Pod**：Pod退出了，此类型的Pod不会被创建
  - 无法确保稳定

- **控制器管理的Pod**：在控制器的生命周期里，始终要维持Pod的副本数目

**编程**

- **声明式编程（Deployment）**：侧重于定义想要什么，然后告诉计算机让它帮你实现
  - apply > create  
- **命令式编程（ReplicaSet）**：侧重于如何实现程序，把实现过程按逻辑一步步写出
  -  create > apply      



## Namespace

命名空间是**对一组资源和对象的抽象集合**，比如可以用来**将系统内部的对象划分为不同的项目组或者用户组**

- 常见的pod、service、replicaSet和deployment等都是属于某一个namespace的（默认是default）
- node, persistentVolumes等则不属于任何namespace

```bash
# 查询所有namespace
kubectl get namespace
# 创建namespace
kubectl create namespacensname
# 删除namespace
kubectl delete namespacensname
```

**删除namespace**

- 删除一个namespace会**自动删除所有属于该namespace的资源**
- **default和kube-system 命名空间不可删除**
- PersistentVolumes是不属于任何namespace的，但PersistentVolumeClaim是属于某个特定namespace的
- Events是否属于namespace取决于产生events的对象

## Service

Service是对**一组提供相同功能的Pods的抽象**，并为他们**提供一个统一的入口**

- 借助Service应用可以方便的**实现服务发现与负载均衡**，并实现应用的**零宕机升级**

- Service**通过标签（label）来选取后端Pod**，一般配合ReplicaSet或者Deployment来保证后端容器的正常运行

- Service有四种类型，**默认是ClusterIP**

  - ClusterIP：默认类型，**自动分配一个仅集群内部可以访问的虚拟IP**

  - NodePort：在ClusterIP基础上为Service**在每台机器上绑定一个端口**
    - 可以通过`NodeIP:NodePort`访问该服务

  - LoadBalancer：在NodePort的基础上，借助cloud provider创建一个**外部的负载均衡器**，并将**请求转发**到NodeIP:NodePort

  - ExternalName：将服务通过DNS CNAME记录方式**转发到指定的域名**

> 也可以将已有的服务**以Service的形式加入到Kubernetes集群中**来


## 其他组件

- CoreDns：可以为集群中的SVC创建一个域名IP的对应关系解析
- DashBoard：给Kubernetes集群提供一个B/S结构访问体系
- **IngressController**：Ingress可以实现七层代理
  - 官方只能实现四层代理

- Federatiom：提供一个可以跨集群中心多Kubernetes统一管理功能
- **Prometheus**：提供Kubernetes集群的监控能力
- **Efk**：提供Kubernetes集群日志统一分析介入平台

# 部署Kubernetes集群

集群类型

- 单Master集群
- 多Master集群

先部署一套单Master架构（3台），再扩容为多Master架构（4台或6台）

**单Master服务器规划**

| **角色**   | **IP**        | **组件**                                                     |
| ---------- | ------------- | ------------------------------------------------------------ |
| k8s-master | 192.168.31.71 | kube-apiserver，kube-controller-manager，kube-scheduler，etcd |
| k8s-node1  | 192.168.31.72 | kubelet，kube-proxy，docker，etcd                            |
| k8s-node2  | 192.168.31.73 | kubelet，kube-proxy，docker，etcd                            |

**服务器要求**

- 2核CPU，2G内存，30G硬盘

- 集群中所有机器之间网络互通

- 可以访问外网，需要拉取镜像（NAT）

- 禁止swap分区

| **软件**   | **版本**               |
| ---------- | ---------------------- |
| 操作系统   | CentOS7.x_x64 （mini） |
| 容器引擎   | Docker CE 19           |
| Kubernetes | Kubernetes v1.20       |

**搭建方式**

- **kubeadm**：K8s 部署工具，快速部署Kubernetes集群
  - `kubeadm init`：创建Master节点
  - `kubeadm join`：将Node节点加入集群`kubeadm join <Master节点IP:端口>`
- **二进制**：手动部署每个组件，组成Kubernetes集群

> Kubeadm 降低部署门槛，但屏蔽了很多细节，遇到问题很难排查
>
> 使用二进制包部署Kubernetes 集群更容易可控，也利于后期维护

## 安装虚拟机

**新建虚拟机**

- 稍后安装OS
- 2个处理器，2个内核
- 2G内存
- 100G磁盘
- 磁盘存储为单个文件

**虚拟机设置**

- CD/DVD选择CentOS镜像
- 网络适配器选择NAT模式
- 安装位置默认
- 最小化安装
- 密码：空格

**地址分配**

| 节点         | IP             |
| ------------ | -------------- |
| k8s-master01 | 192.168.192.31 |
| k8s-node01   | 192.168.192.32 |
| k8s-node02   | 192.168.192.33 |

## 系统初始化

**查看NAT网关**

1. VMware-编辑-虚拟网络编辑器-更改设置
2. 选择NAT模式的网卡，点击NAT设置
3. 查看网关IP：192.168.192.2

![NAT](Kubernetes.assets/NAT.png)

```bash
# 设置IP地址
vi /etc/sysconfig/network-scripts/ifcfg-ens33
# 修改
BOOTPROTO=static
ONBOOT=yes
# 增加 
# 三个节点分别是31,32,33
IPADDR=192.168.192.31
NETMASK=255.255.255.0
GATEWAY=192.168.192.2

# 设置DNS
vi /etc/resolv.conf
# 增加
nameserver 192.168.192.2
# 重启
service network restart
# 验证
ping www.baidu.com

# 安装wget
yum install wget
```

![撰写栏](Kubernetes.assets/撰写栏.png)

同时写入三个节点

```bash
# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 添加hosts
cat >> /etc/hosts << EOF
192.168.192.31 k8s-master01
192.168.192.32 k8s-node01
192.168.192.33 k8s-node02
EOF

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 重启

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# 生效
sysctl --system  

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com

## 安装docker

Kubernetes 默认CRI（容器运行时）为Docker，因此先安装Docker

​```bash
# 下载Docker
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 安装指定版本
yum -y install docker-ce-18.06.1.ce-3.el7

# 设置开机启动
systemctl enable docker && systemctl start docker

# 验证
docker --version

# 设置仓库
cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF

# 重启docker
systemctl restart docker

# 验证
docker info

# 添加yum源
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

## 安装k8s组件

- kubeadm
- kubectl
- kubelet

```bash
# 指定版本
yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0

# 设置开机启动
systemctl enable kubelet
```

## 部署Master节点

- apiserver-advertise-address：当前节点IP
- image-repository：阿里云镜像
- kubernetes-version：指定版本

> 默认拉取镜像地址k8s.gcr.io国内无法访问，指定阿里云镜像仓库地址

```bash
# 在Master节点上执行
kubeadm init \
--apiserver-advertise-address=192.168.192.31 \ 
--image-repository registry.aliyuncs.com/google_containers \ 
--kubernetes-version v1.18.0 \ 
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16
```

验证安装成功

- 提示成功信息**initialized successfully**
- 提示接下来步骤

![kubeadm init](Kubernetes.assets/kubeadm init.png)

查看下载的镜像

![master images](Kubernetes.assets/master images.png)

按提示操作

```bash
# 在master上运行
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 查看节点,只有master节点,状态是NotReady
kubectl get nodes
```

## 添加Node节点

按提示操作

> 默认的token：`sha256:94a3812eff925d99fcc31b54febe76863bc0606da49fc09def8b97ea0e4b7227`有效期是24h，过期需要重新创建

```bash
# 在node上运行
kubeadm join 192.168.192.31:6443 --token 1f2a01.p9gda3tufqr2299x \
    --discovery-token-ca-cert-hash sha256:94a3812eff925d99fcc31b54febe76863bc0606da49fc09def8b97ea0e4b7227
    
# 查看节点,有master节点和node节点,状态都是NotReady
kubectl get nodes

# 重新创建token
kubeadm token create --print-join-command
```

## 部署CNI网络插件

节点状态NotReady：缺少网络组件

> 默认镜像地址无法访问，sed命令修改为docker hub镜像仓库。

```bash
# 下载网络插件配置
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 查看pods
kubectl get pods -n kube-system
```

状态变为Ready

![集群状态](Kubernetes.assets/集群状态.png)

[K8S应用FLANNEL失败解决INIT:IMAGEPULLBACKOFF](https://www.cnblogs.com/pyxuexi/p/14288591.html)

## 测试k8s集群

在Kubernetes集群中创建一个pod验证是否正常运行

在master节点上运行

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
```

访问：http://192.168.192.32:30561  

![验证](Kubernetes.assets/验证.png)



## Harbor

>  企业级 Docker 私有仓库

安装底层需求

- Python应该是应该是2.7或更高版本
- Docker引擎应为引擎应为1.10或更高版本
- Docker Compose需要为需要为1.6.0或更高版本

```bash
# 添加假证书
vim /etc/docker/daemon.json
# 中括号后加逗号，
"insecure-registries": ["[https://hub.atguigu.com](https://hub.atguigu.com/)"]

# 重启
systemctl restart docker
mv docker-compose /usr/local/bin
chmod a+x /usr/local/bin/docker-compose

# 解压
tar -zxvf harbor-offline-installer-v1.2.0.tgz
mv harbor /usr/local/
cd /usr/local/harbor
mkdir -p /data/cert

openssl genrsa -des3 -out server.key 2048
# 创建密码：123456

openssl req -new -key server.key -out server.csr
# 输入密码：123456
# 输入国家CN
# 输入城市BJ
# 默认城市BJ
# 组织atguigu

# 备份私钥
cp server.key server.key.org
# 退出密码
openssl rsa -in server.key.org -out server.key
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
mkdir  /data/cert
chmod -R 777 /data/cert
docker ps -a

# 访问hub.atguigu.com
# 登陆：admin
# 密码：Harbor12345
# 访问尝试
docker login https://hub.atguigu.com
# 下载镜像
docker pull wangyanglinux/myapp:v1
# 命令名称
# 规范：docker tag SOURCE_IMAGE[:TAG] hub.atguigu.com/library/IMAGE[:TAG]
# 把SOURCE_IMAGE[:TAG]和IMAGE[:TAG]替换掉
docker tag wangyanglinux/myapp:v1 hub.atguigu.com/library/myapp:v1
# 上传镜像到仓库
docker  push hub.atguigu.com/library/myapp:v1
# 链接私有仓库
kubectl run nginx-deployment --image=hub.atguigu.com/library/myapp:v1 --port=80 --replicas=1
# replicas标识副本数目，必须保持为1
# 删除
kubectl delete pod +名称
# 修改数目
kubectl scale --replicas=3 deployment/nginx-deployment
kubectl get deployment
kubectl get node -o wide

# 查看帮助文档
kubectl expose --help
# 访问模板
kubectl expose deployment nginx --port=80 --target-port=8000
# 访问服务的80端口，访问容器的8000端口（Create a service for an nginx deployment, which serves on port 80 and connects to the containers on port 8000）
# 修改虚拟ip为外网ip
kubectl edit svc nginx-deployment
# 将type里的ClusterIP改为NodePort
netstat -anpt | grep :30000
```

## Rancher

- Rancher的集群管理基于角色的访问控制策略，策略管理和工作负载等功能在导入集群中可用
- 对于除K3s集群外的所有导入的 Kubernetes 集群，必须在Rancher外部编辑集群的配置，需要自己在集群中修改Kubernetes组件的参数、升级Kubernetes版本以及添加或删除节点
- Rancher中不能配置或扩展导入的集群

**搭建Rancher环境**

> ERROR: Rancher must be ran with the --privileged flag when running outside of Kubernetes

在浏览器中访问：https://192.168.192.31，首次访问会提示设置admin管理员密码

```bash
sudo docker run --privileged -d  --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:stable
```



# YAML

Kubernetes集群中对**资源管理**和**资源对象编排部署**都可以通过YAML文件（声明样式文件）来解决

- 可以把需要对资源对象的操作编辑到YAML格式文件中，把这种文件叫做**资源清单文件**
- 通过kubectl命令直接使用资源清单文件就可以实现对大量的资源对象进行编排部署

**书写格式**

- YAML是一种**标记语言**，**以数据做为中心**，而不是以标记语言为重点
- YAML是一个可读性高，用来表达数据序列的格式

**基本语法**

* 使用**空格做为缩进**
  * 缩进的空格数目不重要，只要相同层级的元素左侧**对齐即可**
  * 低版本**缩进时不允许使用Tab键**，只允许使用空格

* **使用`#`标识注释**
  * 从`#`字符一直到行尾，都会被解释器忽略

**数据结构**

- **对象**：键值对集合

  - 映射（mapping）

  - 哈希（hashes）

  - 字典（dictionary）


- **数组**：一组按次序排列的值

  - 序列（sequence）

  - 列表（list）


# kubectl

`kubectl command type name flags`

- comand：指定**对资源执行的操作**
  - `create`，`get`，`describe`和`delete`

- type：指定**资源类型**
  - 大小写敏感，可以是单数、复数和缩略的形式
  - `pod`，`pods`和`po`

- name：**指定资源的名称**
  - 大小写敏感，省略名称会显示所有的资源

- flags：**指定可选的参数**
  - 用`-s`或者`–server`参数指定API server 的地址和端口




# Kuboard

**在Kuboard中集群概览的展示形式**

- 上层：运行于计算资源与存储资源上的名称空间（应用）

- 下层：计算资源、存储资源并列




**在Kuboard中名称空间的展示形式：以微服务参考分层架构的形式，将所有的微服务分为如下几层：**

- 展现层：终端用户访问的 Web 应用
- API网关层：Spring Cloud Gateway / Zuul /Kong 等接口网关
- 微服务层：Spring Boot 微服务，或 PHP / Python 实现的微服务
- 持久层：MySQL 数据库等（开发及测试环境里将MySQL部署于K8s可以极大地降低环境维护的任务量）
- 中间件层
  - 消息队列
  - 服务注册 Eureka / Zookeeper / Consul
- 监控层
  - Prometheus + Grafana
  - Pinpooint



# 资源清单

```bash
# 报错时候查看
kubectl describe pod podname

# 查看日志信息，pod里面单个容器
kubectl logs podname

# 查看日志信息，pod里面多个容器，指定容器名
kubectl logs podname -c containername

# 查看pod详细信息
kubectl get pod -o wide

# 查看pod详细信息，追踪状态
kubectl get pod -w
```

## 创建pod

使用`pod.yaml`文件创建pod

- 格式要对齐，同一级别的对象要放在同一列，不用tab

```bash
# 拉取服务镜像
docker pull busybox

# 编写init容器的yaml文件
vim ini-pod.yaml
```

`ini-pod.yaml`文件

```yaml
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
     command: [ 'sh', '-c', 'echo The app is running! && sleep 3600' ]
  initContainers:
  -  name: init-myservice
     image: busybox
     command: [ 'sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;' ]
  - name: init-mydb
     image: busybox
     command: [ 'sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;' ]
```

操作pod

```bash
# 创建pod
kubectl create -f ini-pod.yaml

# 查看pod信息
kubectl describe pod myapp-pod

# 查看单容器的pod的服务日志
kubectl logs myapp-pod

# 查看多容器的pod的服务日志
kubectl logs myapp-pod -c init-myservice

# 删除pod
kubectl delete pod myapp-pod

# 删除所有的pod
kubectl delete pod --all
```

## 创建pod服务

创建pod里面对应的服务的配置文件

`vim myservice.yaml`

```yaml
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

`vim mydb.yaml`

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

进入pod中的服务

```bash
# 进入单容器pod
kubectl exec -it readiness-httpget-pod --/bin/sh

# 进入多容器pod需要指明容器
kubectl exec -it readiness-httpget-pod -c containername --/bin/sh
```

## 检测探针



# 资源调度

```bash
# 将node标志为不可调度
kubectl cordon nodename
# 将node标志为可调度
kubectl uncordon nodename
```

## 污点

taint

使用kubectl taint命令可以**给某个Node节点设置污点**

- Node被设置上污点之后就和Pod之间存在了一种相斥的关系，可以让Node拒绝Pod的调度执行，甚至将Node已经存在的Pod驱逐出去

- 每个污点的组成：`key=value:effect`

- 当前taint effect支持三个选项

  - `NoSchedule`：表示k8s将不会将Pod调度到具有该污点的Node上

  - `PreferNoSchedule`：表示k8s将尽量避免将Pod调度到具有该污点的Node上

  - `NoExecute`：表示k8s将不会将Pod调度到具有该污点的Node上，同时会将Node上已经存在的Pod驱逐出去

```bash
# 为node0设置不可调度污点
kubectl taint node node0 key1=value1:NoShedule

# 将node0上key值为key1的污点移除
kubectl taint node node0 key-

# 为kube-master节点设置不可调度污点
kubectl taint node node1 node-role.kubernetes.io/master=:NoSchedule

# 为kube-master节点设置尽量不可调度污点
kubectl taint node node1 node-role.kubernetes.io/master=PreferNoSchedule
```

## 容忍

Tolerations

设置了污点的Node将根据taint的effect和Pod之间产生互斥的关系，Pod将在一定程度上不会被调度到Node上。

可以在Pod上设置容忍（Toleration），设置了容忍的Pod将可以容忍污点的存在，可以被调度到存在污点的Node上



# 存储卷

Volume

默认情况下**容器的数据是非持久化**的，容器消亡以后数据也会跟着丢失，所以**Docker提供了Volume机制以便将数据持久化存储**

Kubernetes提供了更强大的Volume机制和插件，解决了**容器数据持久化**以及**容器间共享数据**的问题

Kubernetes存储卷的生命周期与Pod绑定

- 容器挂掉后Kubelet再次**重启容器时，Volume的数据依然还在**
- **Pod删除时，Volume才会清理**
- 数据是否丢失取决于具体的Volume类型
  - emptyDir的数据会丢失，而PV的数据则不会丢

目前Kubernetes主要支持以下Volume类型

- emptyDir：Pod存在，emptyDir就会存在，容器挂掉不会引起emptyDir目录下的数据丢失，但是pod被删除或者迁移，emptyDir也会被删除
- hostPath：hostPath允许挂载Node上的文件系统到Pod里面去
- NFS：Network File System，网络文件系统，Kubernetes中通过简单地配置就可以挂载NFS到Pod中，而NFS中的数据是可以永久保存的，同时NFS支持同时写操作
- glusterfs：同NFS一样是一种网络文件系统，Kubernetes可以将glusterfs挂载到Pod中，并进行永久保存
- cephfs：一种分布式网络文件系统，可以挂载到Pod中，并进行永久保存
- subpath：Pod的多个容器使用同一个Volume时，会经常用到
- secret：密钥管理，可以将敏感信息进行加密之后保存并挂载到Pod中
- persistentVolumeClaim：用于将持久化存储（PersistentVolume）挂载到Pod中

## 持久化存储卷

PersistentVolume（PV） 

PersistentVolume是集群之中的一块**网络存储资源**

- PersistentVolume（PV） 和PersistentVolumeClaim（PVC）提供了方便的持久化卷
  - PV提供网络存储资源，PVC请求存储资源并将其挂载到Pod中

**访问模式**

PV的访问模式（accessModes）有三种

- ReadWriteOnce（RWO）：是最基本的方式，可读可写，但只支持被单个Pod挂载。

- ReadOnlyMany（ROX）：可以以只读的方式被多个Pod挂载

- ReadWriteMany（RWX）：可以以读写的方式被多个Pod共享

不是每一种存储都支持这三种方式。**在PVC绑定PV时通常根据两个条件来绑定**，一个是存储的大小，另一个就是访问模式

**回收策略**

PV的回收策略（persistentVolumeReclaimPolicy）有三种

- Retain：不清理，保留Volume，需要手动清理
- Recycle：删除数据，即`rm -rf /thevolume/*` 
  - 只有NFS和HostPath支持
- Delete：删除存储资源

# 无状态应用

Deployment

一般情况不需要手动创建Pod实例，而是**采用更高一层的抽象或定义来管理Pod**

针对无状态类型的应用，Kubernetes使用Deloyment的Controller对象与之对应，典型的应用场景包括

- 定义Deployment来创建Pod和ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续Deployment

Deployment和ReplicaSet

- **使用Deployment来创建ReplicaSet**
  - ReplicaSet在后台创建pod，检查Pod启动状态
- 当执行更新操作时，会创建一个新的ReplicaSet
  - Deployment会按照控制的速率将pod从旧的ReplicaSet移动到新的ReplicaSet中

```bash
# 查找Deployment
kubectl get deployment --all -namespaces 

# 查看某个Deployment
kubectl describe deployment deploymentname

# 编辑Deployment定义
kubectl edit deployment deploymentname

# 删除某Deployment
kubectl delete deployment deploymentname

# 扩缩容操作，修改Deployment下的Pod实例个数
kubectl scale deployment/deploymentname --replicas=2 
```

# 有状态应用

StatefulSet

StatefulSet是Kubernetes为了有状态服务而设计，其应用场景包括

- **稳定的持久化存储**：Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
- **稳定的网络标志**：Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现
- **有序部署和有序扩展**：Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次进行操作
  - 在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态，基于init containers来实现
- **有序收缩**：按标号有序删除

StatefulSet支持**两种更新策略**

- OnDelete：当.spec.template更新时，并不立即删除旧的Pod，而是等待用户手动删除这些旧Pod后自动创建新Pod
  - 默认的更新策略，兼容v1.6版本的行为
- RollingUpdate：当.spec.template 更新时，自动删除旧的Pod并创建新Pod替换，在更新时这些Pod是按逆序的方式进行
  - 依次删除、创建并等待Pod变成Ready状态才进行下一个Pod的更新

# 守护进程集

DaemonSet 

DaemonSet保证在**特定或所有Node节点上都运行一个Pod实例**，常用来部署一些集群的日志采集、监控或者其他系统管理应用，典型的应用包括

- 日志收集：比如fluentd，logstash等
- 系统监控：比如Prometheus Node Exporter，collectd等
- 系统程序：比如kube-proxy，kube-dns，glusterd，ceph，ingress-controller等

**DaemonSet会忽略Node的unschedulable状态**，有两种方式来指定Pod只运行在指定的Node节点上

- `nodeSelector`：只调度到匹配指定label的Node上
- `nodeAffinity`：功能更丰富的Node选择器，比如支持集合操作
- `podAffinity`：调度到满足条件的Pod所在的Node上

目前支持两种策略

- OnDelete：默认策略，更新模板后，只有手动删除了旧的Pod后才会创建新的Pod
- RollingUpdate：更新DaemonSet模版后，自动删除旧的Pod并创建新的Pod

# Ingress

Kubernetes中的负载均衡主要有两种机制

- **Service**：使用Service提供**集群内部的负载均衡**
  - Kube-proxy负责将service请求负载均衡到后端的Pod中
- **Ingress Controller**：使用Ingress提供**集群外部的负载均衡**

**Service和Pod的IP仅可在集群内部访问**，集群外部的请求需要通过负载均衡转发到service所在节点暴露的端口上，然后再由kube-proxy通过边缘路由器将其转发到相关的Pod

Ingress可以给service提供**集群外部访问的URL**、负载均衡、HTTP路由等

- 配置Ingress规则需要部署一个Ingress Controller，它监听Ingress和service的变化，并根据规则配置负载均衡并提供访问入口

- 常用的ingress controller

  - nginx

  - traefik

  - Kong

  - Openresty

# HPA

Horizontal Pod Autoscaling：水平伸缩

**HPA可以根据CPU、内存使用率或应用自定义metrics自动扩展Pod数量** 

- 支持replication controller、deployment和replica set

- 控制管理器**默认每隔30s查询metrics的资源使用情况**

  - 可以通过`--horizontal-pod-autoscaler-sync-period`修改

- 支持三种metrics类型

- - 预定义metrics（比如Pod的CPU）以利用率的方式计算
  - 自定义的Pod metrics，以原始值（raw value）的方式计算
  - 自定义的object metrics

- 支持两种metrics查询方式

  - Heapster
  - 自定义的REST API

- 支持多metrics
