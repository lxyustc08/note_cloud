- [Kubernetes架构](#kubernetes%e6%9e%b6%e6%9e%84)
  - [Master组件](#master%e7%bb%84%e4%bb%b6)
    - [kube-apiserver](#kube-apiserver)
    - [etcd](#etcd)
    - [kube-scheduler](#kube-scheduler)
    - [kube-controller-manager](#kube-controller-manager)
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
+ 
