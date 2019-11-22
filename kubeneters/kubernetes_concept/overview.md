- [Kubernetes概述](#kubernetes%e6%a6%82%e8%bf%b0)
  - [Kubernetes是什么](#kubernetes%e6%98%af%e4%bb%80%e4%b9%88)
  - [Kubernetes可以做什么](#kubernetes%e5%8f%af%e4%bb%a5%e5%81%9a%e4%bb%80%e4%b9%88)
  - [Kubernetes不会做什么](#kubernetes%e4%b8%8d%e4%bc%9a%e5%81%9a%e4%bb%80%e4%b9%88)
# Kubernetes概述
## Kubernetes是什么
Kubernetes：一个可移植、可扩展、开源的容器管理平台，管理容器化的工作负载及服务；Kubernetes促进<span style="border-bottom:4px solid red; font-weight: bold">陈述化</span>的配置及自动化。
## Kubernetes可以做什么  
Kubernetes提供了分布式系统弹性部署运行机制；对于应用而言，Kubernetes仔细考虑了应用的弹性伸缩、故障迁移(failover)，提供部署模式(deployment patterns)描述，提供的详细服务如下：
+ 服务发现及负载均衡，Service discovery and Load Balancing  
  + Kubernetes通过容器名称或者容器IP公开容器
  + 如果某容器流量(traffic)过高，Kubernetes将流量或负载均衡到其他容器上，保证该部署运行状态稳定
+ 存储编排， Storage orchestration
  + Kubernetes支持多种存储后端自动挂载，包括但不限于本地存储、云存储等
+ 自动发布及自动回滚，Automated rollouts and rollbacks
  + 通过Kubernetes，用户可定义部署容器状态，<span style="border-bottom: 2px solid red; font-weight: bold">Kubernetes以受控的方式自动将实际状态调度调整至用户定义的状态</span>
+ 自动打包，Automatic bin packing
  + Kubernetes根据用户指定的CPU及内存配置在Kubernetes集群中调度容器
+ 自愈，Self-healing
  + Kubernetes根据用户指定的健康检查规则自动重启失效容器，替换失效容器，杀掉无响应容器
+ 密钥及配置文件管理，Secret and configuration management
  + Kubernetes支持用户存储及管理加密信息，比如密码、OAuth密钥以及ssh密钥，在不重建容器镜像的前提下，用户可部署及升级密钥及配置文件  
## Kubernetes不会做什么
Kubernetes与PaaS平台不一样，Kubernetes提供了构建开发平台所需的构建块，通过这种方式Kubernetes保留了用户选择及弹性，因此Kubernetes并不会对下列功能做限制：
+ 不限制提供的应用类型，IF an application can run in a container, it should run great on Kubernetes.
+ 不部署代码，不构建应用, Continuous Integration, Delivery, and Depolyment (CI/CD) workflows are determined by organization cultures and preferences as well as technical requirements.
+ 不提供应用层面上的服务, Kubernetes不集成诸如中间件、数据处理框架，数据库，缓存，分布式存储的应用。上述组件可以运行在Kubernetes中，或者通过运行在Kubernetes上的应用访问。
+ 不提供日志、监控以及告警服务，仅提供指标收集及导出机制，供其他应用集成使用。
+ 不提供、不指定特定的配置语言或系统，仅提供<span style="border-bottom: 2px solid red; font-weight: bold">declarative API</span>，该API可被任意格式的declarative specifications定位使用；
+ 不提供、不采用任何comprehensive machine configuration, maintenance, management, or self-healing systems;
+ Kubernetes不仅是一个编排系统，<span style="border-bottom: 2px solid red; font-weight: bold">事实上他消除了编排的需求</span>：
  + 对于编排技术而言，需要定义一个工作流：首先做A，然后B，然后C
  + 相反的，对于Kubernetes而言，其包含了a set of independent, composable control processes that <span style="border-bottom: 2px solid red; font-weight: bold">continuously drive the current state towards the provided desired state</span>
  + <span style="border-bottom: 2px solid red; font-weight: bold">Kubernetes不关注如何从A到C，中心化的控制也不要求提供A到C的路径</span>
  + 这使得Kubernetes更为易用，更为强力，更为鲁棒，更为弹性，更易于扩展