- [Kubernetes架构](#kubernetes%e6%9e%b6%e6%9e%84)
  - [Master组件](#master%e7%bb%84%e4%bb%b6)
    - [kube-apiserver](#kube-apiserver)
    - [etcd](#etcd)
    - [kube-scheduler](#kube-scheduler)
    - [kube-controller-manager](#kube-controller-manager)
    - [cloud-controller-manager](#cloud-controller-manager)
  - [Node组件](#node%e7%bb%84%e4%bb%b6)
    - [kubelet](#kubelet)
    - [kube-proxy](#kube-proxy)
    - [Container Runtime](#container-runtime)
  - [Addons](#addons)
    - [DNS](#dns)
    - [Web UI (Dashboard)](#web-ui-dashboard)
    - [Container Resource Monitoring](#container-resource-monitoring)
    - [Cluster-level logging](#cluster-level-logging)
  - [Kubernetes架构xmind](#kubernetes%e6%9e%b6%e6%9e%84xmind)
# Kubernetes架构
![Alt text](components-of-kubernetes.png "Kubernetes组件")
+ Master Node
  + manages the worker nodes and the pods in the cluster
+ Worker Node
  + host the pods that are the components of the application
## Master组件
Master组件提供集群的控制平面。
>Master components make global decisions about the cluster (for example, scheduling), and they detect and respond to cluster events (for example, starting up a new pod when a deployment’s replicas field is unsatisfied)  

<span style="border-bottom: 2px solid red; font-weight: bold">默认情况下Master节点上不运行任何容器</span>

### kube-apiserver
kube-apiserver是Kubernetes控制平面的前端，对外公布Kubernetes API接口
>The API server is a component of the Kubernetes control plane that exposes the Kubernetes API  

kube-apiserver被设计为水平扩展，可通过部署多个实例来扩展api server的服务能力
### etcd
etcd作为Kubernetes后端存储，用于存储集群数据，关于etcd备份详看[etcd备份](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)  
关于etcd详细信息参考[etcd官方文档](https://etcd.io/docs/v3.4.0/)
### kube-scheduler
该组件监控新创建的未分配运行节点的pods，选择一个nodes，将该pods调度到该nodes上，调度时考虑的因素如下：
+ individual and collective resource requirements
+ hardware/software/policy constraints
+ affinity and anti-affinity specifications 亲和性
+ data locality 数据位置
+ inter-workload interference and deadlines
### kube-controller-manager
运行在master节点上的运行控制器的组件，<span style="border-bottom: 2px solid red; font-weight: bold">逻辑上而言，每个控制器是一个独立的进程</span>，但为了降低复杂度，<span style="border-bottom: 2px solid red; font-weight: bold">所有控制器目前被打包成一个软件，且仅运行一个进程上</span>，控制器如下：
+ Node Controller: 当node宕机时，由Node Controller负责通知及响应
+ Replication Controller： 对于每个Replication controller object，由其负责维护正确的pods数目
+ Endpoint Controller： 部署Endpoint object
+ Service Account & Token Controllers： 负责新的namespace，由其负责创建默认账号及API访问token

### cloud-controller-manager
cloud-controller-manager负责运行与底层云供应商交互的控制组件，<span style="border-bottom: 2px solid red; font-weight: bold">该功能从Kubernetes 1.6版本引入</span>。
> <span style="color: #DC143C; font-weight: bold; font-family: 'Helvetica'; font-size: 15px">1.6版本后的Kubernetes引入原生混合云支持，从1.6版本后云服务商的代码可以与Kubernetes独立迭代发展</span>

cloud-controller-manager仅运行云服务商提供的控制器控制流程循环，<span style="border-bottom: 2px solid red; font-weight: bold">在本地集群kube-controller-manager中需要禁止cloud-controller-manager中的控制器循环</span>，通过将参数```--cloud-provider```设置为```external```，即可实现上述目的。对于下述控制器而言，与云服务商存在依赖关系：
+ Node Controller： 检查云服务提供商提供的节点终止响应后是否被删除
+ Router Controller： 设置云服务提供商提供的路由基础设施
+ Service Controller： 创建、更新以及删除云服务商的load balance
+ Volume Controller: 创建、绑定、挂载卷，并与云服务商交互以编排卷资源

## Node组件
Node组件运行于各个节点上，负责运行pods，并提供Kubernetes运行环境
### kubelet
运行于集群的各个节点上，确保容器运行于Pods中。  
kubelet以一系列```PodSpecs```为输入，```PodSpecs```提供了多种机制；kubelet确保容器按照```PodSpecs```中规定的健康运行。<span style="border-bottom: 2px solid red; font-weight: bold">kubelet并不管理非Kubernetes创建的容器</span>
### kube-proxy
kube-proxy是运行在各个节点上的网络代理程序，用于实现Kubernetes service相关的概念。  
kube-proxy维护各节点上的网络规则，通过网络规则，可以实现Pods的网络通信（网络通信包括集群内部或者集群外部）  
通常而言，kube-proxy使用当前OS的网络包过滤层，比如IPtables，如果当前集群中无可用的网络过滤层，则由kube-proxy自己进行流量转发。
### Container Runtime
容器运行环境，用于支撑容器运行。  
目前Kubernetes支持多种容器运行环境：[Docker](https://www.docker.com/)、[containerd](https://containerd.io/)、[cri-o](https://cri-o.io/)、[rktlet](https://github.com/kubernetes-incubator/rktlet)以及一切实现[Kubernetes CRI（Container Runtime interface）](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)的容器运行环境。

## Addons
Addons插件：利用Kubernetes资源([DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)等)实现集群的相关特性，由于本质上是实现集群的特性，因此Addons插件运行的Kubernetes namespace均为```kube-system```。几个典型的Addons如下。
### DNS  

<span style="border-bottom: 2px solid red; font-weight: bold">尽管DNS是以插件的形式提供，但对于Kubernetes集群而言，是必不可少的</span>   

集群的DNS主要为Kubernetes service服务  
所有由Kubernetes创建的容器均包含Kubernetes集群中的DNS服务  
### Web UI (Dashboard)
Kubernetes集群的基于Web的可视化管理界面
### Container Resource Monitoring
提供容器运行状态数据记录，记录存储在中心化的数据库中；并提供可视化UI供数据查看
### Cluster-level logging
存储集群容器运行日志，日志存储在中心数据库中，提供搜索/浏览接口


## Kubernetes架构xmind
![Alt Text](Kubernetes&#32;components.png)
