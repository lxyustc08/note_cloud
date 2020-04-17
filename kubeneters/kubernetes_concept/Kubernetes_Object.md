- [Kubernetes Object基本内涵](#kubernetes-object%e5%9f%ba%e6%9c%ac%e5%86%85%e6%b6%b5)
  - [Object Spec and States](#object-spec-and-states)
  - [描述一个Kubernetes对象](#%e6%8f%8f%e8%bf%b0%e4%b8%80%e4%b8%aakubernetes%e5%af%b9%e8%b1%a1)
  - [必要的域Fields](#%e5%bf%85%e8%a6%81%e7%9a%84%e5%9f%9ffields)
- [Kubernetes对象管理](#kubernetes%e5%af%b9%e8%b1%a1%e7%ae%a1%e7%90%86)
  - [Imperative commands](#imperative-commands)
  - [Imperative object configuration](#imperative-object-configuration)
  - [Declarative object configuration](#declarative-object-configuration)
- [对象标识](#%e5%af%b9%e8%b1%a1%e6%a0%87%e8%af%86)
  - [Name](#name)
  - [UID](#uid)
- [Namespaces](#namespaces)
  - [Namespaces的使用场景](#namespaces%e7%9a%84%e4%bd%bf%e7%94%a8%e5%9c%ba%e6%99%af)
  - [初始的Kubernetes Namespace](#%e5%88%9d%e5%a7%8b%e7%9a%84kubernetes-namespace)
  - [Namespace与DNS](#namespace%e4%b8%8edns)
  - [不是所有Kubernetes对象均在namespace中](#%e4%b8%8d%e6%98%af%e6%89%80%e6%9c%89kubernetes%e5%af%b9%e8%b1%a1%e5%9d%87%e5%9c%a8namespace%e4%b8%ad)
- [Labels and Selectors](#labels-and-selectors)
  - [Motivation](#motivation)
  - [Syntax and character set](#syntax-and-character-set)
    - [key-键语法](#key-%e9%94%ae%e8%af%ad%e6%b3%95)
    - [value-值语法](#value-%e5%80%bc%e8%af%ad%e6%b3%95)
  - [Label selectors](#label-selectors)
    - [Equality-based requirement](#equality-based-requirement)
    - [Set-based requirement](#set-based-requirement)
  - [API](#api)
    - [List and WATCH filtering](#list-and-watch-filtering)
    - [API对象中的引用](#api%e5%af%b9%e8%b1%a1%e4%b8%ad%e7%9a%84%e5%bc%95%e7%94%a8)
      - [Service and ReplicationController](#service-and-replicationcontroller)
      - [支持set-based requirements的资源/对象](#%e6%94%af%e6%8c%81set-based-requirements%e7%9a%84%e8%b5%84%e6%ba%90%e5%af%b9%e8%b1%a1)
      - [节点选择操作](#%e8%8a%82%e7%82%b9%e9%80%89%e6%8b%a9%e6%93%8d%e4%bd%9c)
- [Annotation](#annotation)
  - [Syntax and character set](#syntax-and-character-set-1)
    - [Annotation key语法](#annotation-key%e8%af%ad%e6%b3%95)
- [Fields Selectors](#fields-selectors)
  - [Supported fields](#supported-fields)
  - [Supported operators](#supported-operators)
  - [Chained selectors](#chained-selectors)
  - [Multiple Resource types](#multiple-resource-types)
- [Recommended Labels](#recommended-labels)
  - [Labels建议](#labels%e5%bb%ba%e8%ae%ae)
  - [应用及应用的实例](#%e5%ba%94%e7%94%a8%e5%8f%8a%e5%ba%94%e7%94%a8%e7%9a%84%e5%ae%9e%e4%be%8b)
  - [一些例子](#%e4%b8%80%e4%ba%9b%e4%be%8b%e5%ad%90)
    - [简单的无状态服务](#%e7%ae%80%e5%8d%95%e7%9a%84%e6%97%a0%e7%8a%b6%e6%80%81%e6%9c%8d%e5%8a%a1)
    - [使用数据库的网页应用](#%e4%bd%bf%e7%94%a8%e6%95%b0%e6%8d%ae%e5%ba%93%e7%9a%84%e7%bd%91%e9%a1%b5%e5%ba%94%e7%94%a8)
# Kubernetes Object基本内涵
**Kubernetes Object是Kubernetes系统中持久化的记录**，<span style="border-bottom: 2px solid red; font-weight: bold">代表着Kubernetes集群的某一状态</span>，特别地，可以用来描述如下状态：
+ 那个容器化的应用正在运行，运行在哪个节点上
+ 适用于应用的资源
+ 围绕应用行为的策略，比如重启策略、升级策略以及失效保障策略  
  
<span style="border-bottom: 2px solid red; font-weight: bold">Kubernetes对象本质上是一个`预期的记录`</span>，当创建一个Kubernetes对象后，本质上你告诉Kubernetes集群需要将工作负载表现为何种状态，也即集群的`desired state`。  

通过[Kubernetes API](https://kubernetes.io/docs/reference/using-api/api-concepts/)管理Kubernetes对象，包括创建、修改及删除等操作。也可通过`kubectl`工具或者[客户端库](https://kubernetes.io/docs/reference/using-api/client-libraries/)管理Kubernetes对象  
## Object Spec and States
每个Kubernetes对象由两个嵌套的对象域(object fields)组成：`object spec`与`object status`
+ object spec，用户提供，描述Kubernetes对象的预期状态`desire state`，该object field必须提供，不得缺少；
+ object status，Kubernetes对象的实际状态`actual state`，由Kubernetes system进行实现和更新；  

<span style="border-bottom: 2px solid red; font-weight: bold">在任何时刻，Kubernetes Control Plane积极控制Kubernetes对象，使Kubernetes对象状态趋近于预期状态`desire state`</span>

## 描述一个Kubernetes对象
创建Kubernetes对象时需要提供Kubernetes对象的Object Spec及其他基本信息如对象名称等，若采用Kubernetes API进行，则在Request body中以json格式提供上述信息；若采用`kubectl`工具，则通过yaml文件提供，由`kubectl`工具将yaml文件中的信息转换为API所需的格式。一个典型的yaml文件如下：
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

## 必要的域Fields  

<span style="border-bottom: 2px solid red; font-weight: bold">使用`kubectl`工具创建Kubernetes对象</span>时必须提供的域如下:  

+ apiVersion: 创建Kubernetes对象时使用的API版本
+ kind： 创建Kubernetes对象时指定的对象类别
+ metadata： 用于帮助用户区分对象的数据，包括`name` string、`UID`以及可选项`namespace`
+ spec: 即对象的`desire state`  

# Kubernetes对象管理
总共有3种方式管理Kubernetes对象：
+ Imperative commands
  + 直接使用kubectl命令
+ Imperative object configuration
  + 使用Kubernetes对象描述文件，该描述文件必须包含完整的Kubernetes对象描述
+ Declarative object configuration
  + 用户无需向`kubectl`工具指定配置文件的操作，由`kubectl`自动识别
  + 不同的操作使用不同的配置文件

3种不同方式的区别如下表所示。

<table>
    <tr>
        <th>Management technique</th>
        <th>Operations on</th>
        <th>Recommended environment</th>
        <th>Supported writers</th>
        <th>Learning curve 学习曲线</th>
    </tr>
    <tr>
        <td>Imperative commands</td>
        <td>Live objects</td>
        <td>Development projects</td>
        <td>1+</td>
        <td>Lowest</td>
    </tr>
    <tr>
        <td>Imperative object configuration</td>
        <td>Individual files</td>
        <td>Production projects</td>
        <td>1</td>
        <td>Moderate</td>
    </tr>
    <tr>
        <td>Declarative object configuration</td>
        <td>Directories of files</td>
        <td>Production projects</td>
        <td>1+</td>
        <td>Highest</td>
    </tr>
</table>

## Imperative commands
这种方式直接对集群中处于活动状态对象(live objects)进行操作。用户直接将操作作为命令`kubectl`的参数或标志。  
这种方式是**最为简单**的Kubernetes对象操作方式，提供一次性的任务操作，不记录历史配置。  
优势：
+ Commands are simple, easy to learn and easy to remember.
+ Commands require only a single step to make changes to the cluster.  

劣势：
+ Commands do not integrate with change review processes.
+ Commands do not provide an audit trail associated with changes.
+ Commands do not provide a source of records except for what is live.
+ Commands do not provide a template for creating new objects.

## Imperative object configuration
这种方式使用配置文件作为`kubectl`命令输入，配置文件中必须包含Kubernetes对象的完整定义，<span style="border-bottom: 2px solid red; font-weight: bold">除配置文件外用户需要向`kubectl`命令提供Kubernetes对象的操作及可选参数。</span> 

优势：
+ 对象配置文件可以使用各类代码版本控制程序进行管理
+ 对象配置文件可以引入代码审查之类的质量控制过程
  > 可以作为测试点
+ 对象配置文件某种程度上也引入了模板`template`机制，利用该机制可以更为方面创建新的应用
  
  > <span style="color: #DC143C; font-weight: bold; font-family: 'Helvetica'; font-size: 15px">1. 以Kubernetes，Mesosphere Marathon为代表的容器平台模板应该更偏向于应用层面上的描述  
  > 2. Kubernetes 和 Mesosphere Marathon之间还是存在一定的差别，不可混为一谈</span>

劣势：
+ 需要理解Kubernetes对象描述文件的Scheme
+ 增加了编写Kubernetes对象描述文件的工作量

与下面要介绍的declarative object configuration相比。  
优势：
+ Imperative object configuration易于理解，操作简单
+ 从Kubernetes 1.5之后Imperative object configuration更为成熟*mature*

劣势：
+ Imperative object configuration最好以文件为操作对象而非文件夹
+ 更新活动Kubernetes对象时必须将更新反应在配置文件中，这样的话下次更新后，本次更新情况信息丢失；

一个例子，Kubernetes对象配置文件nginx.yaml:  
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
创建Kubernetes对象
```terminal
# kubectl create -f nginx.yaml
```
删除Kubernetes对象
```terminal
# kubectl delete -f nginx.yaml
```
配置文件更新后，更新Kubernetes对象
```terminal
# kubectl replace -f nginx.yaml
```
关于Imperative object configuration，参考[信息](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/)

## Declarative object configuration
这种同样使用配置文件作为输入，但与Imperative object configuration不同在于Declarative object configuration方式在使用时无需向`kubectl`工具提供具体的Kubernetes对象操作，由`kubectl`工具自动识别。  
这种方式工作在文件夹中，不同的Kubernetes对象的不同操作需要单独的一个文件。与Imperative object configuration相比。  
优势：
+ 所有针对活动Kubernetes对象的更新信息将会被保留，甚至未更新至配置文件中的信息也会保留
+ 对文件夹操作具有更好的支持度  
劣势：
+ 出问题后难以调试
+ 使用diffs进行局部更新引入复杂的合并及分支操作  

例子，假定所有的Kubernetes对象配置文件存储在`configs`文件夹中，更新时先使用`diff`查看区别，再使用`apply`应用
```terminal
# kubectl diff -f configs/
# kubectl apply -f configs/
```
针对文件递归操作
```terminal
# kubectl diff -f -R configs/
# kubectl apply -f -R configs/
```
关于Declarative object configuration，参考[信息](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)

# 对象标识
每个Kuberntes对象都有一个标识，有两种标识方式：`Name`或`UID`。对于`Name`而言，每类对象中的对象`Name`唯一，对于`UID`而言，每个Kubernetes集群中的对象`UID`唯一。

## Name
由客户端提供的一串资源URL，例如`/api/v1/pods/some-name`  
在某一特定类型范围内，Kubernetes对象名唯一。  
Kubernetes对象名支持253字符，允许的对象名为`数字`,`小写字母`, `-`, `.`。  
一个例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
```

## UID
有Kubernetes系统生成的对象唯一标识符，Kubernetes UID也是一种UUIDs

# Namespaces
本质上而言，Namespaces是Kubernetes的虚拟集群抽象，<span style="border-bottom: 2px solid red; font-weight: bold">一个namespace对应一个虚拟集群</span>

<span style="border-bottom: 2px solid red; font-weight: bold">关于namespace集群抽象的补充：</span>

1. Namespace提供了一种在不同的组中操作互相隔离的`kubernetes objects`对象的机制；
2. Namespace并未提供任何隔离机制，若某一**Namespace A**中的`kubernetes objects`知道另一个**Namespace B**中的`kuberentes objects`的IP地址，且IP地址可达，则可正常通信；
3. 可通过网络隔离设计来实现Namespace间的隔离，这种是由网络解决方案来决定的，而非Namespace本身的机制决定的。

## Namespaces的使用场景
Namespace设计意图：解决集群中分属不同的团队、项目的多个用户使用需求。Namespace的一些特性：
+ Namespace提供了命名域支持，同一Namespace中Kubernetes对象名称唯一，但不同的Namespace之间，Kubernetes对象名称可以重复
+ 基于Namespace可实现集群的资源分割，可查阅[资源配额](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
+ 后续版本中，将默认同一Namespace内部访问控制权限一致

<span style="border-bottom: 2px solid red; font-weight: bold">不建议也无必要使用Namespace进行轻量化的资源区分，比如不同的软件版本之类，对于类似的需求可采用</span>`labels`<span style="border-bottom: 2px solid red; font-weight: bold">进行区分即可</span>。

关于Namespace相关操作参见[namespace创建删除](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/#creating-a-new-namespace), [namespace查看、请求操作设置](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)。

## 初始的Kubernetes Namespace
Kubernetes集群创建后会自动创建3个namespace分别为`default`，`kube-system`，`kube-public`
+ default， 未指定Kubernetes对象的namespace时，对象默认的namespace
+ kube-system， 由Kubernetes系统创建的Kubernetes对象的namespace
+ kube-public， 自动创建，且可被所有用户（包括未授权用户）读取，（create automatically and is readable by all users）
  > This namespace is mostly reserved for cluster usage, in case that some resources should be visible and readable publicly throughout the whole cluster. The public aspect of this namespace is only **a convention, not a requirement**.

## Namespace与DNS
在创建一个`Service`后，Kubernetes创建对应的DNS记录，形式如`<service-name>.<namespace-name>.svc.cluster.local`，对于同一个namespace中的容器（**该容器是由Kubernetes创建**）而言，可以自动通过`<service-name>`定位到该服务。  
若要跨namespace寻址，则需要使用FQDN也即绝对DNS。

## 不是所有Kubernetes对象均在namespace中
namespace不在namespace中，底层资源，诸如nodes节点，持久化存储卷也不属于任何一个namespace，可通过下述两条命令判断哪些资源在namespace中，哪些资源不在namespace中。
> <span style="border-bottom: 2px solid red; font-weight: bold">在namespace中的对象可逻辑隔离，不在namespace中的对象不逻辑隔离，或者说是“全局的”</span>

列出在namespace中的对象
```terminal
# kubectl api-resources --namespaced=true
NAME                        SHORTNAMES   APIGROUP                    NAMESPACED   KIND
bindings                                                             true         Binding
configmaps                  cm                                       true         ConfigMap
endpoints                   ep                                       true         Endpoints
events                      ev                                       true         Event
limitranges                 limits                                   true         LimitRange
persistentvolumeclaims      pvc                                      true         PersistentVolumeClaim
pods                        po                                       true         Pod
podtemplates                                                         true         PodTemplate
replicationcontrollers      rc                                       true         ReplicationController
resourcequotas              quota                                    true         ResourceQuota
secrets                                                              true         Secret
serviceaccounts             sa                                       true         ServiceAccount
services                    svc                                      true         Service
controllerrevisions                      apps                        true         ControllerRevision
daemonsets                  ds           apps                        true         DaemonSet
deployments                 deploy       apps                        true         Deployment
replicasets                 rs           apps                        true         ReplicaSet
statefulsets                sts          apps                        true         StatefulSet
localsubjectaccessreviews                authorization.k8s.io        true         LocalSubjectAccessReview
horizontalpodautoscalers    hpa          autoscaling                 true         HorizontalPodAutoscaler
cronjobs                    cj           batch                       true         CronJob
jobs                                     batch                       true         Job
leases                                   coordination.k8s.io         true         Lease
networkpolicies                          crd.projectcalico.org       true         NetworkPolicy
networksets                              crd.projectcalico.org       true         NetworkSet
events                      ev           events.k8s.io               true         Event
ingresses                   ing          extensions                  true         Ingress
ingresses                   ing          networking.k8s.io           true         Ingress
networkpolicies             netpol       networking.k8s.io           true         NetworkPolicy
poddisruptionbudgets        pdb          policy                      true         PodDisruptionBudget
rolebindings                             rbac.authorization.k8s.io   true         RoleBinding
roles                                    rbac.authorization.k8s.io   true         Role
```
列出不在namespace中的对象
```terminal
# kubectl api-resources --namespaced=false
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
componentstatuses                 cs                                          false        ComponentStatus
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumes                 pv                                          false        PersistentVolume
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
tokenreviews                                   authentication.k8s.io          false        TokenReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
bgpconfigurations                              crd.projectcalico.org          false        BGPConfiguration
bgppeers                                       crd.projectcalico.org          false        BGPPeer
blockaffinities                                crd.projectcalico.org          false        BlockAffinity
clusterinformations                            crd.projectcalico.org          false        ClusterInformation
felixconfigurations                            crd.projectcalico.org          false        FelixConfiguration
globalnetworkpolicies                          crd.projectcalico.org          false        GlobalNetworkPolicy
globalnetworksets                              crd.projectcalico.org          false        GlobalNetworkSet
hostendpoints                                  crd.projectcalico.org          false        HostEndpoint
ipamblocks                                     crd.projectcalico.org          false        IPAMBlock
ipamconfigs                                    crd.projectcalico.org          false        IPAMConfig
ipamhandles                                    crd.projectcalico.org          false        IPAMHandle
ippools                                        crd.projectcalico.org          false        IPPool
runtimeclasses                                 node.k8s.io                    false        RuntimeClass
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
csidrivers                                     storage.k8s.io                 false        CSIDriver
csinodes                                       storage.k8s.io                 false        CSINode
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```

# Labels and Selectors  
Labels是附加在Kubernetes对象上的键值对，提供对象标签。一般使用Labels组织Kubernetes对象，并提供Kubernetes对象子集的选择机制。  
Labels可在Kubernetes对象创建时添加，也可在Kubernetes对象创建完成后按需求添加或修改。
> Labels提供了额外的Kubernetes对象管理机制。<span style="border-bottom: 2px solid red; font-weight: bold">在某种程度上提供了后续Kubernetes对象的统计分析机制支持，提供了高效查询及查看机制</span>

每个Kubernetes对象可以拥有多个Labels，这些Labels组成一个集合，**集合中的键值对的`key`值必须唯一**，一个例子：
```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```  

## Motivation
Labels允许用户将自己的组织结构以一种松散的方式映射到Kubernetes对象中，避免客户端在本地存储这种映射关系。  
<span style="border-bottom: 2px solid red; font-weight: bold">通过引入Labels机制，可以在固定的基础设施上进行灵活的业务管理层级调整，避免死板的基础设施影响业务的灵活性调整。</span>

## Syntax and character set
如前所述Labels是键值对集合。  
### key-键语法
合理的键语法如下：  
```
valid_key => (optional)prefix/name
```  
name:  
+ 不长于63字符
+ 以字母数字`[a-z0-9A-Z]`、短划线`-`、下划线`_`打头;
+ 以字母数字`[a-z0-9A-Z]`、短划线`-`、下划线`_`结尾；
+ 中间若干位字母数字`[a-z0-9A-Z]`；

prefix：
+ 可选项
+ 若指定，则必须为一个DNS的子域，也即一系列由点`.`分割的DNS Labels组成的序列
  + 长度不超过253字符
+ 未指定，默认该Labels为用户私有，不公开
  + 由Kubernetes自动创建的组件，如kube-scheduler、kube-controller-manager、kube-apiserver、kubectl，当他们附加标签至终端用户对象时，必须指定Labels的prefix；
  + 标签前缀kubernetes.io/及k8s.io/为保留Labels，供Kubernetes核心使用
### value-值语法
+ 不超过63字符
+ 为空
+ 不为空
  + 以字母数字`[a-z0-9A-Z]`、短划线`-`、下划线`_`打头；
  + 以字母数字`[a-z0-9A-Z]`、短划线`-`、下划线`_`结尾；
  + 中间若干位字母数字`[a-z0-9A-Z]`；  

典型例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
```  
## Label selectors
Labels不具备独特性，从设计上来说，应该让不同的Kubernetes对象具备同一个Labels。  
对于Label selector而言，其提供了一种对象集合选择方式，Label selector是<span style="border-bottom: 2px solid red; font-weight: bold">Kubernetes的core grouping primitive</span>。  

目前Kubernetes API支持两种selector，`equality-based`及`set-based`，<span style="border-bottom: 2px solid red; font-weight: bold">一个label选择器由多个用逗号分开`requirements`组成。对于这些`requirements`而言，选择器必须同时满足这些需求，因此可以将逗号理解为逻辑运算符`AND(&&)`。</span>  

<span style="border-bottom: 2px solid red; font-weight: bold">The semantics of empty or non-specified selectors are dependent on the context, and API types that use selectors should document the validity and meaning of them.</span>  
 
> For some API types, such as ReplicaSets, the label selectors of two instances must not overlap within a namespace, or the controller can see that as conflicting instructions and fail to determine how many replicas should be present.  

> For both equality-based and set-based conditions <span style="border-bottom: 2px solid red; font-weight: bold">there is no logical OR (||) operator. Ensure your filter statements are structured accordingly.</span>

### Equality-based requirement  
值相等关系选择器，有三种形式`=`,`==`,`!=`，前面两种表明相等关系，后面一种表明不等关系；比如下列所述：
```
environment = production
tier != frontend
```
即选择Kubernetes对象中的key为environment，value等于production，key为tier，value不等于frontend的对象。  
另一个列子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```
这个例子展示了equality-based requirement的另一个用法：决定pods节点选择标准，例子中的含义即为选择具备tesla p100 加速能力的节点。

### Set-based requirement
基于集合从属关系选择，有三种关系`in`,`notin`,`exists`，对于`exists`而言，其仅对于key有效，一些例子：
```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```
前两个易于理解，第三个指存在key为partition的Label的Kubernetes对象，第四个指不存在key为partition的Label的Kubernetes对象。  
## API
### List and WATCH filtering
通过`LIST`与`WATCH`操作，可以指定label selector从而筛选对象集合。上述操作可通过query参数进行，例子：  

equality-based requirements
```
?labelSelector=environment%3Dproduction,tier%3Dfrontend
```  
set-based requirements
```
?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29
```

同样的也可通过kubectl工具进行对象选择，例子：

equality-based requirements
```
# kubectl get pods -l environment=production,tier=frontend
```
set-based requirements
```
# kubectl get pods -l 'environment in (production),tier in (frontend)'
```

### API对象中的引用
部分Kubernetes对象使用label selector指定其需要的资源，比如`services`及`replicationcontrollers`

#### Service and ReplicationController
对于`services`及`replicationcontrollers`对象而言，<span style="border-bottom: 2px solid red; font-weight: bold">仅可使用equality-based requirements选择器指定其所需资源</span>；指定时使用yaml或json格式均可，例子：  

json格式
```json
"selector": {
    "component" : "redis",
}
```
yaml格式
```yaml
selector:
    component: redis
```

#### 支持set-based requirements的资源/对象
对于`Job`，`Deployment`，`Replica Set`，`Daemon Set`这些较新的资源/对象而言，他们支持set-based requirements，一个例子：
```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```
`matchLabels`：形如{key, value}的键值对，数据结构为map   
`matchExpressionts`: 由`key`, `operator`, `values`组成的匹配表达式，数据结构为List  

`matchLabels`与`matchExpressions`可以相互转换，比如：  
```yaml
matchLabels:
  component: redis
```
等价于
```yaml
matchExpressions:
  - {key: component, operator: In, values: [redis]}
```

#### 节点选择操作
通过labels选择`nodes`本质上即为调度`pods`到某些运行`nodes`上，详见[node selection](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)  


# Annotation
annotation除了label之外的可附加在Kubernetes对象上的元数据，Annotation提供了一种注释解释机制，方便用户使用。Annotation与label类似也是由键值对组成的。一个例子：
```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```
Annotation的一些作用：
+ Fields managed by a declarative configuration layer. Attaching these fields as annotations distinguishes them from default values set by clients or servers, and from auto-generated fields and fields set by auto-sizing or auto-scaling systems.
+ Build, release, or image information like timestamps, release IDs, git branch, PR numbers, image hashes, and registry address.
+ Pointers to logging, monitoring, analytics, or audit repositories.
+ Client library or tool information that can be used for debugging purposes: for example, name, version, and build information.
+ User or tool/system provenance information, such as URLs of related objects from other ecosystem components.
+ Lightweight rollout tool metadata: for example, config or checkpoints.
+ Phone or pager numbers of persons responsible, or directory entries that specify where that information can be found, such as a team web site.
+ Directives from the end-user to the implementations to modify behavior or engage non-standard features.

## Syntax and character set
Annotation是键值对。  

### Annotation key语法
valid_annotion_keys => (optional)preix/name  
name：
+ name不超过63字符
+ 以字母数字`[a-z0-9A-Z]`、短划线`-`、下划线`_`打头;
+ 以字母数字`[a-z0-9A-Z]`、短划线`-`、下划线`_`结尾；
+ 中间若干位字母数字`[a-z0-9A-Z]`； 

prefix：
+ 可选项
+ 若指定，则必须为一个DNS的子域，也即一系列由点`.`分割的DNS Labels组成的序列
  + 长度不超过253字符
+ 未指定，默认该Annotation key为用户私有，不公开
  + 由Kubernetes自动创建的组件，如kube-scheduler、kube-controller-manager、kube-apiserver、kubectl，当他们注释附加至终端用户对象时，必须指定Annotation的prefix；
  + 标签前缀kubernetes.io/及k8s.io/为保留Annotation prefix，供Kubernetes核心使用  

一个例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
```

# Fields Selectors
`Field selectors`允许用户根据<span style="border-bottom: 2px solid red; font-weight: bold">资源的域的值</span>选择Kubernetes资源。  
一些例子：
+ meta-name=my-service
+ metadata.namespace!=default
+ status.phase=Pending  

使用`kubectl`工具可以较为方便的指定feild selectors,比如：
```terminal
# kubectl get pods --field-selector status.phase=Running
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-dqsj9   1/1     Running   0          2d1h
nginx-deployment-54f57cf6bf-zxmzg   1/1     Running   0          2d1h
```
本质上而言，Fields selector是Kubernetes资源的过滤器，未指定选择器/过滤器，等价于所有的条件都可以，比如：
```terminal
# kubectl get pods -A
default                nginx-deployment-54f57cf6bf-dqsj9            1/1     Running   0          2d2h
default                nginx-deployment-54f57cf6bf-zxmzg            1/1     Running   0          2d2h
kube-system            calico-kube-controllers-6b64bcd855-pdlcc     1/1     Running   2          9d
kube-system            calico-node-898ql                            1/1     Running   2          9d
kube-system            calico-node-bvz5d                            1/1     Running   2          8d
kube-system            calico-node-vwwf6                            1/1     Running   2          8d
kube-system            coredns-5644d7b6d9-4955j                     1/1     Running   2          9d
kube-system            coredns-5644d7b6d9-rzb2q                     1/1     Running   2          9d
kube-system            etcd-master                                  1/1     Running   2          9d
kube-system            kube-apiserver-master                        1/1     Running   4          9d
kube-system            kube-controller-manager-master               1/1     Running   2          9d
kube-system            kube-proxy-hhh47                             1/1     Running   2          8d
kube-system            kube-proxy-mq8jx                             1/1     Running   2          9d
kube-system            kube-proxy-nrfdb                             1/1     Running   2          8d
kube-system            kube-scheduler-master                        1/1     Running   2          9d
kubernetes-dashboard   dashboard-metrics-scraper-76585494d8-vm2tf   1/1     Running   2          8d
kubernetes-dashboard   kubernetes-dashboard-b65488c4-99hqs          1/1     Running   2          8d
```
等价于
```terminal
# kubectl get pods --field-selector "" -A
default                nginx-deployment-54f57cf6bf-dqsj9            1/1     Running   0          2d2h
default                nginx-deployment-54f57cf6bf-zxmzg            1/1     Running   0          2d2h
kube-system            calico-kube-controllers-6b64bcd855-pdlcc     1/1     Running   2          9d
kube-system            calico-node-898ql                            1/1     Running   2          9d
kube-system            calico-node-bvz5d                            1/1     Running   2          8d
kube-system            calico-node-vwwf6                            1/1     Running   2          8d
kube-system            coredns-5644d7b6d9-4955j                     1/1     Running   2          9d
kube-system            coredns-5644d7b6d9-rzb2q                     1/1     Running   2          9d
kube-system            etcd-master                                  1/1     Running   2          9d
kube-system            kube-apiserver-master                        1/1     Running   4          9d
kube-system            kube-controller-manager-master               1/1     Running   2          9d
kube-system            kube-proxy-hhh47                             1/1     Running   2          8d
kube-system            kube-proxy-mq8jx                             1/1     Running   2          9d
kube-system            kube-proxy-nrfdb                             1/1     Running   2          8d
kube-system            kube-scheduler-master                        1/1     Running   2          9d
kubernetes-dashboard   dashboard-metrics-scraper-76585494d8-vm2tf   1/1     Running   2          8d
kubernetes-dashboard   kubernetes-dashboard-b65488c4-99hqs          1/1     Running   2          8d
```

## Supported fields
不同的Resource支持的field selector不一样，总体而言，所有的resource类型均支持两个选择器`metadata.name`及`metadata.namespace`。**使用Resource不支持的selector将产生一个错误**。  
例子：

```terminal
# kubectl get ingress --field-selector foo.bar=baz
Error from server (BadRequest): Unable to find "ingresses" that match label selector "", field selector "foo.bar=baz": "foo.bar" is not a known field selector: only "metadata.name", "metadata.namespace"
```

## Supported operators
field selector支持的操作有三种`=`，`==`，`！=`，前两种均表示相等，最后一种表示不相等。  
一个例子
```terminal
# kubectl get services -A --field-selector metadata.namespace!=default
NAMESPACE              NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
kube-system            kube-dns                    ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   9d
kube-system            kubernetes-dashboard        ClusterIP   10.110.84.10     <none>        443/TCP                  8d
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.104.193.119   <none>        8000/TCP                 8d
kubernetes-dashboard   kubernetes-dashboard        ClusterIP   10.98.131.28     <none>        443/TCP                  8d
```  

## Chained selectors
与Label选择器及其他选择器类似，fields选择器可以以`，`逗号的形式将不同的条件组合起来，`，`也是代表逻辑操作符`&&`,一个例子：
```terminal
# kubectl get pods --field-selector 'status.phase=Running,metadata.namespace=default'
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-dqsj9   1/1     Running   0          2d3h
nginx-deployment-54f57cf6bf-zxmzg   1/1     Running   0          2d3h
```  
选择运行状态为Running，namespace为default的pods。

## Multiple Resource types
可使用selector同时选择多种资源，一个例子：
```
# kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```  

# Recommended Labels
使用对象标签时，若遵循一定的规范，可以帮助不同的Kubernetes对象更好地被管理，有助于用户使用。

使用包含对象标签的metadata时**建议**以应用概念为中心进行组织。<span style="border-bottom: 2px solid red; font-weight: bold">Kubernetes不是一个服务平台，对于Kubernetes而言，其并未给出应用application的正式含义；取而代之的是将应用application视为一个非正式的概念，需要通过metadata去描述。</span>

> 对于Kubernetes而言，基本概念为Kubernetes对象，Kubernetes Objects, Kubernetes对象与application之间无任何关系。

## Labels建议
建议所有共享的Labels均已前缀`app.kubernetes.io`打头，前面内容已经介绍，无前缀的Labels均为私有Labels。

建议的Labels key及相关含义如下表所示：

<table>
  <tr>
    <th>Key</th>
    <th>Description</th>
    <th>Example</th>
    <th>Type</th>
  </tr>
  <tr>
    <td>app.kubernetes.io/name</td>
    <td>The name of the application</td>
    <td>mysql</td>
    <td>string</td>
  </tr>
  <tr>
    <td>app.kubernetes.io/instance</td>
    <td>A unique name identifying the instance of an application</td>
    <td>wordpress-abcxzy</td>
    <td>string</td>
  </tr>
  <tr>
    <td>app.kubernetes.io/version</td>
    <td>The current version of the application (e.g., a semantic version, revision hash, etc.)</td>
    <td>5.7.21</td>
    <td>string</td>
  </tr>
  <tr>
    <td>app.kubernetes.io/component</td>
    <td>The component within the architecture</td>
    <td>database</td>
    <td>string</td>
  </tr>
  <tr>
    <td>app.kubernetes.io/part-of</td>
    <td>The name of a higher level application this one is part of</td>
    <td>wordpress</td>
    <td>string</td>
  </tr>
  <tr>
    <td>app.kubernetes.io/managed-by</td>
    <td>The tool being used to manage the operation of an application</td>
    <td>helm</td>
    <td>string</td>
  </tr>
</table>  

一个例子
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
```

## 应用及应用的实例
一个应用可以拥有多个实例，对于应用而言应用名称与应用实例名称分开管理。比如一个应用WordPress有应用名称标签`app.kubernetes.io/name=wordpress`，但同时其有一个实例名称标签`app.kubernetes.io/instance=wordpress-abcxyz`。

对于同一个应用而言，其实例名称应该唯一

## 一些例子
### 简单的无状态服务
使用`Deployment`与`Service`对象部署的无状态服务。  
`Deployment`监督pods运行该应用，描述如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

`Service`对外开放该应用
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

### 使用数据库的网页应用
考虑使用Mysql的WordPress应用，该应用假设由Helm部署。  

部署WordPress的描述
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```
使用`Service`对外开放应用描述
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

作为`StatefulSet`的Mysql描述：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```  

对外开放Mysql `Service`描述：
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```
