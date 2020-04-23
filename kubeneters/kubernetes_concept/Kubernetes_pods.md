- [Kubernetes Pods](#kubernetes-pods)
  - [Inter-Pod network](#inter-pod-network)
  - [Pods一种理解角度](#pods%e4%b8%80%e7%a7%8d%e7%90%86%e8%a7%a3%e8%a7%92%e5%ba%a6)
- [Pods的合适组织方式](#pods%e7%9a%84%e5%90%88%e9%80%82%e7%bb%84%e7%bb%87%e6%96%b9%e5%bc%8f)
  - [Pods中运行多个Container的建议设计模式](#pods%e4%b8%ad%e8%bf%90%e8%a1%8c%e5%a4%9a%e4%b8%aacontainer%e7%9a%84%e5%bb%ba%e8%ae%ae%e8%ae%be%e8%ae%a1%e6%a8%a1%e5%bc%8f)
- [Container Probe](#container-probe)
    - [Creating effective liveness probes](#creating-effective-liveness-probes)
      - [了解liveness probe到底检查的是什么](#%e4%ba%86%e8%a7%a3liveness-probe%e5%88%b0%e5%ba%95%e6%a3%80%e6%9f%a5%e7%9a%84%e6%98%af%e4%bb%80%e4%b9%88)
      - [保持检查器轻量化](#%e4%bf%9d%e6%8c%81%e6%a3%80%e6%9f%a5%e5%99%a8%e8%bd%bb%e9%87%8f%e5%8c%96)
      - [无需定义RETRY LOOPS次数](#%e6%97%a0%e9%9c%80%e5%ae%9a%e4%b9%89retry-loops%e6%ac%a1%e6%95%b0)
      - [liveness probe的限制](#liveness-probe%e7%9a%84%e9%99%90%e5%88%b6)

# Kubernetes Pods

Pods是kubernetes最基本的`kubernetes objects`，kubernetes采用pods这一概念的原因：

1. container本身提供了强力隔离机制，而且一个container中只运行一个进程，但某些时候需要一组container完成特定任务，此时希望这一组contianer之间可以进行`namespace`共享；
2. kubernetes pods提供了这种机制，通过配置Kubernetes pods内的container容器之间共享部分`namespaces`，如`Network`，`UTS`，`IPC`甚至于`PID`（默认情况下不开启）；
3. kubernetes pods提出的目的就在于获得隔离和整体的平衡~balance；
4. kubernetes pods内的container之间的文件系统通过volume共享；

同一个Pod中的所有`Container`共享同一个`network namespace`，同一个`IP地址`，同一个`port空间`。<span style="border-bottom: 2px solid red; font-weight: bold">同一个Pod中的Container中的进程在使用网络端口时需要考虑端口占用问题，避免出现端口冲突。</span>

同样的，同一个Pod中的`Container`共享网络接口，因此他们共享同一个`loopback network interface`，因此可以通过**localhost**进行通信。

## Inter-Pod network

Pod之间的网络是一个扁平结构**flat network**，各Pod拥有独立的IP地址，位于同一个二层网络中，之间不存在NAT转换。

![Alt Text](kubernetes_pod_inter_network.png)

## Pods一种理解角度

1. ***在逻辑上而言，pods是一种logical hosts，举止行为表现为物理机或者虚拟机***；
2. ***运行在pods中的进程，就像物理机或者虚拟机中的进程一样，唯一不同的在于，pods中运行的进程每个均运行在独立的Container中***；

# Pods的合适组织方式

如前所述，一个Pod可以视为一个主机，但由于其轻量化特性，并且从应用松耦合角度出发，应该尽量将应用分散至不同的Pods中，而非全部集中在同一个Pod里。遵循两个原则：

1. Spliting Multi-Tier Apps into Multiple Pods
2. Spliting into Multiple Pods to Enable Individual Scaling

## Pods中运行多个Container的建议设计模式

Pods中运行多个Container的建议模式是，应用由一个主进程与多个协助进程组成，如下图所示。

![Alt Text](kubernetes_multiple_container_run_in_a_pod.png)

多个Container运行在同一个Pods中的考虑出发点可基于如下出发点进行考虑：

+ Do they need to be run together or can they run on different hosts?
+ Do they represent a single whole or are they independent components?
+ Must they be scaled together or individually?

![Alt text](kubernetes_thinking_multiple_container_run_in_a_single_pod.png)

# Container Probe

Container Probe是一个探测器，由`kubelet`执行探测活动，在执行`Probe`时，`kubelet`调用由Container实现的Handler。对于Handler而言，可分为三类：

1. ExecAction
   + 执行指定的命令，当命令成功执行并返回0时，探测被认为成功
2. TCPSocketAction
   + 在Container的IP地址的特定端口上执行TCP链接检测，当端口处于打开状态时，探测被认为成功
3. HTTPGetAction
   + 在Conatiner的IP地址的特定端口及路径上执行HTTP GET请求，如果相应状态码大于等于200或小于400，则探测被认为成功

每个`Probe`由三种结果：

1. Success: Container通过检测
2. Failure: Container未通过检测
3. Unknown: 探测失败，无需采取行动

对于处于运行状态的`container`而言，kubelet可选择执行及响应3类probes：

+ `livenessProbe`: Indicates whether the Container is running. If the liveness probe fails, the kubelet kills the Container, and the Container is subjected to its restart policy. If a Container does not provide a liveness probe, the default state is `Success`.
+ `readinessProbe`: Indicates whether the Container is ready to service requests. If the readiness probe fails, the endpoints controller removes the Pod’s IP address from the endpoints of all Services that match the Pod. The default state of readiness before the initial delay is `Failure`. If a Container does not provide a readiness probe, the default state is `Success`.
+ `startupProbe`: Indicates whether the application within the Container is started. All other probes are disabled if a startup probe is provided, until it succeeds. If the startup probe fails, the kubelet kills the Container, and the Container is subjected to its restart policy. If a Container does not provide a startup probe, the default state is `Success`.

<span style="border-bottom: 2px solid red; font-weight: bold">在生产环境下，总是应该设置liveness probe，在不设置liveness probe的情况下，kubernetes无法知道你的应用是否处于存活状态</span>

### Creating effective liveness probes

#### 了解liveness probe到底检查的是什么

liveness probe可提供应用层面上的健康检查，这比无liveness probe情况下单纯检查contianer的运行状态来的高效，同时liveness probe在应用层面上执行健康检查时可指定健康检查的路径`/path`

> <span style="border-bottom: 2px solid red; font-weight: bold">指定的健康检查路径`/path`不应该要求进行验证，否则liveness probe将由于无法通过验证导致探测失败，最终引起无限的容器重启。</span>

在设置liveness probe时需要明确的是，liveness probe应该探测的是应用的内部因素导致的潜在风险，而无需考虑应用外部因素。

> 比如，前端服务器的liveness probe不应该在后端数据库连接失败时返回错误，因为这可能是由数据库失效导致的，在这种情况下重启前端服务器毫无作用

#### 保持检查器轻量化

liveness probe不应该被设置为占用过多的计算资源，同时不应该占用过多的时间完成健康检查过程

> <span style="border-bottom: 2px solid red; font-weight: bold">liveness probe本身占用的CPU配额被计算到container的CPU配额中，因此若liveness probe占用过多的CPU时间，势必影响到container的运行效能</span>

<span style="border-bottom: 2px solid red; font-weight: bold">特别的，若container中运行的是Java应用程序，此时更应该使用`HTTPGetAction`而不建议使用`ExecGetAction`，因为每次进行`ExecGetAction`检查则将导致创建一个全新的JVM，这将大大占用container的配额，影响container的使用效率。</span>

#### 无需定义RETRY LOOPS次数

虽然可以定义liveness probe检查失败时的重试次数，但建议无需改变默认值

#### liveness probe的限制

正如前述所示，liveness probe是由`kubelet`负责执行探测的，若某一节点失效，此时该节点上的`kubelet`失效，此时liveness probe作用机制将失效，需要kubernetes `control plane`介入，此时需要通过一系列controllers控制器来创建相关Pods才可以实现nodes失效后，该Nodes上的pods承担的任务在其他nodes上重建
