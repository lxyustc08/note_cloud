- [Kubernetes Metric Server](#kubernetes-metric-server)
  - [Introduce kubernetes metric server](#introduce-kubernetes-metric-server)
  - [Metrics server use cases](#metrics-server-use-cases)
  - [Metrics server requirements](#metrics-server-requirements)
  - [Metrics server installation](#metrics-server-installation)
  - [Metrics Server的扩展性](#metrics-server的扩展性)

# Kubernetes Metric Server

## Introduce kubernetes metric server

metric servers是Kubernetes内建的用于弹性伸缩的容器数据收集解决方案。

metric servers从kubelet处获取资源metrics，并使用Kubernetes的Metrics API推送给kubernetes，用于Pod资源的横向扩展与纵向扩展。

+ 横向扩展 [Horizontal Pod Autoscalar](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/): The Horizontal Pod Autoscaler automatically scales the number of Pods in a `replication controller`, `deployment`, `replica set` or `stateful set` based on observed CPU utilization.
  + **Horizontal Pod Autoscalar does not apply to objects that can't be scaled, for example, DaemonSets**
+ 纵向扩展 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) (VPA): frees the users from necessity of setting up-to-date resource limits and requests for the containers in their pods. When configured, it will set the requests automatically based on usage and thus allow proper scheduling onto nodes so that appropriate resource amount is available for each pod.

## Metrics server use cases

Metrics server可被用于下列场合：

+ CPU/memory based horizonal autoscaling
+ Automatically adjusting/suggesting resources needed by containers

Metrics server不可被用于下列场合：

+ Non-Kubernetes clusters
+ An accurate source of resource usage metrics
+ Horizonal autoscaling based on other resources than CPU/Memory.

## Metrics server requirements

+ Metrics server must be reachable from kube-apiserver
+ The kube-apiserver must be correctly configured to enable an aggregation layer
+ Nodes must have kubelet authorization configured to match Metrics Server configuration
+ Container runtime must implement a [container metrics RPCs](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/cri-container-stats.md)

## Metrics server installation

Metrics Server使用Kubernetes配置文件描述部署，配置文件地址[如下](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)

1. 确认部署文件中的镜像
   
   ```yaml
   image: k8s.gcr.io/metrics-server/metrics-server:v0.4.2
   ```

2. 在具有代理设置的节点上拉取镜像
   
   ```
   docker pull k8s.gcr.io/metrics-server/metrics-server:v0.4.2
   ```

3. 分别拉取上述镜像的arm64架构以及amd64架构，并将其上传至本地镜像仓库中
4. 修改部署文件中的镜像，指向本地仓库
5. 修改部署文件中的`--kubelet-preferred-address`选项，内容如下
   
   ```
   --kubelet-preferred-address-types=Hostname,InternalIP,ExternalIP
   ```

6. 添加`--kubelet-insecure-tls`，不采用SSL加密；

7. 使用kubectl edit修改kubernetes集群的dns配置，内容如下，即将所需部署metric的节点域名与IP写入coredns中。
   
   ```
   kubectl edit configmaps -n kube-system coredns
   
    hosts {
            10.10.197.98 master-arm-1
            10.10.197.99 master-arm-2
            10.10.197.97 master-arm-3
            10.10.197.95 worker-amd64-1
            10.10.197.200 worker-amd64-gpuceph-node1
            10.10.197.201 worker-amd64-gpuceph-node2
            10.10.197.202 worker-amd64-gpuceph-node3
            10.10.197.93 worker-arm64-gpu-node1
            10.10.197.94 worker-arm64-gpu-node2
            fallthrough
    }

   ```

8. 使用`kubectl apply`部署metrics server
   
可能存在的问题：

1. liveness probe失败，报错 `Get "https://10.88.6.3:4443/readyz"`失败，可能存在原因为网络插件与Kubernetes版本不兼容，需要升级网络插件。

## Metrics Server的扩展性

> 注：本部分数据来源于Metrics Server的Github页面，其占用的资源与节点的规模如下所示：

对于100个节点的集群而言，实现容器资源metrics的收集，Metrics server需要的资源如下：

+ CPU资源：100m（m为CPU指标的衡量单位，物理意义为所占CPU千分之一，此处100m即为100个千分之一的CPU，也即10%）
+ 内存资源：300MiB

默认配置的Metrics Server的资源上限如下：

|Quantity|Namespace threshold|Cluster threshold|
|:---:|:---:|:---:|
|#Nodes|n/a|100|
|#Pods|7000|7000|
|#Deployments + HPA|100|100|


