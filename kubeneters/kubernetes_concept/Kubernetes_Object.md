- [Kubernetes Object基本内涵](#kubernetes-object%e5%9f%ba%e6%9c%ac%e5%86%85%e6%b6%b5)
  - [Object Spec and States](#object-spec-and-states)
  - [描述一个Kubernetes对象](#%e6%8f%8f%e8%bf%b0%e4%b8%80%e4%b8%aakubernetes%e5%af%b9%e8%b1%a1)
  - [必要的域Fields](#%e5%bf%85%e8%a6%81%e7%9a%84%e5%9f%9ffields)
- [Kubernetes对象管理](#kubernetes%e5%af%b9%e8%b1%a1%e7%ae%a1%e7%90%86)
  - [Imperative commands](#imperative-commands)
  - [Imperative object configuration](#imperative-object-configuration)
  - [Declarative object configuration](#declarative-object-configuration)
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
+ 对处于活动周期的Kubernetes对象进行更新时，会丢失掉前一版本的配置文件

## Declarative object configuration
这种
