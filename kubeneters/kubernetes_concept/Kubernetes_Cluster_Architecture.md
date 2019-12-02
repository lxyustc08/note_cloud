- [Nodes](#nodes)
  - [Node状态](#node%e7%8a%b6%e6%80%81)
    - [Addresses](#addresses)
    - [Conditions](#conditions)
    - [Capacity and Allocatable](#capacity-and-allocatable)
    - [Info](#info)
  - [Management](#management)
    - [Node Controller](#node-controller)

# Nodes
Kubernetes集群工作节点，运行`kubelet`, `kube-proxy`, `container-runtime`,上述三个组件详细介绍参照[Kubernetes node设计介绍](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md#the-kubernetes-node)。

## Node状态
可通过`kubectl describe node ${node-name}`获取node状态信息，例子
```terminal
# kubectl describe node slave-node1
Name:               slave-node1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=slave-node1
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 192.168.122.24/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 172.15.9.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 19 Nov 2019 19:54:46 +0800
Taints:             <none>
Unschedulable:      false
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Fri, 22 Nov 2019 09:19:15 +0800   Fri, 22 Nov 2019 09:19:15 +0800   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Sun, 01 Dec 2019 12:39:32 +0800   Tue, 19 Nov 2019 19:54:46 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Sun, 01 Dec 2019 12:39:32 +0800   Tue, 19 Nov 2019 19:54:46 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Sun, 01 Dec 2019 12:39:32 +0800   Tue, 19 Nov 2019 19:54:46 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Sun, 01 Dec 2019 12:39:32 +0800   Tue, 19 Nov 2019 20:22:28 +0800   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  192.168.122.24
  Hostname:    slave-node1
Capacity:
 cpu:                2
 ephemeral-storage:  10255636Ki
 hugepages-2Mi:      0
 memory:             2041052Ki
 pods:               110
Allocatable:
 cpu:                2
 ephemeral-storage:  9451594122
 hugepages-2Mi:      0
 memory:             1938652Ki
 pods:               110
System Info:
 Machine ID:                 41bf9d733898405ea8d3d2ca275c12b7
 System UUID:                41BF9D73-3898-405E-A8D3-D2CA275C12B7
 Boot ID:                    f118d8c6-86c3-4d3c-bc35-01416cbe1c22
 Kernel Version:             4.15.0-70-generic
 OS Image:                   Ubuntu 18.04.3 LTS
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://18.6.2
 Kubelet Version:            v1.16.1
 Kube-Proxy Version:         v1.16.1
PodCIDR:                     172.15.1.0/24
PodCIDRs:                    172.15.1.0/24
Non-terminated Pods:         (4 in total)
  Namespace                  Name                                   CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                   ------------  ----------  ---------------  -------------  ---
  default                    nginx-deployment-54f57cf6bf-dqsj9      0 (0%)        0 (0%)      0 (0%)           0 (0%)         4d20h
  kube-system                calico-node-vwwf6                      250m (12%)    0 (0%)      0 (0%)           0 (0%)         11d
  kube-system                kube-proxy-hhh47                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         11d
  kubernetes-dashboard       kubernetes-dashboard-b65488c4-99hqs    0 (0%)        0 (0%)      0 (0%)           0 (0%)         11d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                250m (12%)  0 (0%)
  memory             0 (0%)      0 (0%)
  ephemeral-storage  0 (0%)      0 (0%)
Events:              <none>
```
node状态由四部分组成：`Addresses`, `Conditions`, `Capacity and Allocatable`, `Info`。

### Addresses
Node的地址由三部分组成，下述三部分具体值由当前环境确定：
+ Hostname： 节点的网络名称，由节点OS内核给出，可由kubelet命令的`--hostname-override`参数覆盖；
+ ExternalIP： 节点外部网络地址，可外部寻址，不一定存在，根据实际情况显示；
+ InternalIP： 节点内部网络地址，仅可被集群内部寻址；

### Conditions
conditions描述节点运行状态，相关信息及内涵如下表所示。

<table>
    <tr>
        <th>Node Condition</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>Ready</td>
        <td>True if the node is healthy and ready to accept pods, False if the node is not healthy and is not accepting pods, and <span style="border-bottom: 2px solid red; font-weight: bold">Unknown if the node controller has not heard from the node in the last node-monitor-grace-period (default is 40 seconds)</span></td>
    </tr>
    <tr>
        <td>MemoryPressure</td>
        <td>True if pressure exists on the node memory – that is, if the node memory is low; otherwise False</td>
    </tr>
    <tr>
        <td>PIDPressure</td>
        <td>True if pressure exists on the processes – that is, if there are too many processes on the node; otherwise False</td>
    </tr>
    <tr>
        <td>DiskPressure</td>
        <td>True if pressure exists on the disk size – that is, if the disk capacity is low; otherwise False</td>
    </tr>
    <tr>
        <td>NetworkUnavailable</td>
        <td>True if the network for the node is not correctly configured, otherwise False</td>
    </tr>
</table>

节点状态返回值以json格式返回，例子如下：
```json
"conditions": [
  {
    "type": "Ready",
    "status": "True",
    "reason": "KubeletReady",
    "message": "kubelet is posting ready status",
    "lastHeartbeatTime": "2019-06-05T18:38:35Z",
    "lastTransitionTime": "2019-06-05T11:41:27Z"
  }
]
```
对于状态为`Unknown`的节点，<span style="border-bottom: 2px solid red; font-weight: bold">如果`Unknown`或者`False`状态持续时间超过参数`pod-eviction-timeout`指定的时间，运行于该节点上的pods将由Node controller调度，将其删除。</span>
> **Note1:** Kubernetes节点失效处理方式，pod-eviction-timeout为kube-controller-manager传入参数；  
> 
> **Note2:** 默认的eviction time为5分钟；  
> 
> **Note3:** 在某些情况下，node不可达，apiserver无法与node通信，此时删除pods的请求无法通知该节点上的kubelet，位于该node上的pods可能继续运行；  

在Kubernetes 1.5版本之前，node controller会强制删除那些消息无法抵达的Pods。在1.5版本之后，node controller不会自动强制删除这些Pods，除非这些Pods被确认已经停止运行，此时位于unreachable node上的Pods运行状态为`Terminating`或者`Unknown`。对于无法推断node是否永久离开Kubernetes cluster的情况，集群管理员可能需要手动将node删除，删除该node后，该node上的所有Pods对象从apiserver中删除，Pods的名称也被释放。  

1.12版本后，Kubernetes引入`TaintNodeByCondition`机制，该机制引入新的Pods调度模型，<span style="border-bottom: 2px solid red; font-weight: bold">该模型综合考虑节点的Taint及Pod的tolerations确定Pods最终运行的节点。</span>
> 如果pods未设置tolerations，则该pods调度采用旧的调度模型，否则采用新的调度模型

### Capacity and Allocatable
容量`capacity`及可使用量`allocatable`描述node的资源，包括CPU、内存以及该节点支持最大调度Pods数量  
+ 容量`capacity`描述node上总体的资源容量
+ 可使用量`allocatable`描述node上可被普通pods上使用的资源数量
```terminal
Capacity:
 cpu:                2
 ephemeral-storage:  10255636Ki
 hugepages-2Mi:      0
 memory:             2041052Ki
 pods:               110
Allocatable:
 cpu:                2
 ephemeral-storage:  9451594122
 hugepages-2Mi:      0
 memory:             1938652Ki
 pods:               110
```

### Info
描述节点的信息，比如内核版本、Kubernetes版本、容器运行环境版本等。
```terminal
System Info:
 Machine ID:                 41bf9d733898405ea8d3d2ca275c12b7
 System UUID:                41BF9D73-3898-405E-A8D3-D2CA275C12B7
 Boot ID:                    a76aa1bb-dc6d-45c3-8d29-5a2820b45201
 Kernel Version:             4.15.0-70-generic
 OS Image:                   Ubuntu 18.04.3 LTS
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://18.6.2
 Kubelet Version:            v1.16.1
 Kube-Proxy Version:         v1.16.1
```  

## Management
不同于Pods或者service，节点不是由Kubernetes本身创建，其依赖于Cloud provider或者物理节点池或虚拟节点池。因此对于Kubernetes而言，节点的创建动作，<span style="border-bottom: 2px solid red; font-weight: bold">本质上是创建了一个代表节点的对象</span>。节点对象创建完成后，Kubernetes会检查该创建的节点对象是否合法。一个例子:  

节点对象描述
```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```
Kubernetes根据上述描述创建节点对象，然后检查节点运行状态是否健康，是否具备运行pod所需必要的Kubernetes服务，若不符合要求，Kubernetes集群将忽视该对象，直至该对象符合要求。
> **Note:** Kubernetes将保持该节点对象，并持续检查该对象的状态是否合法，直至用户手动将该对象删除。  

目前，有三种方式对Kubernetes节点进行管理`node controller`，`kubectl`，`kubectl`

### Node Controller
`node controller`是Kubernetes主要的节点管理组件，管理各种节点。
