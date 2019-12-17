- [Controlling Access to the Kubernetes API](#controlling-access-to-the-kubernetes-api)
  - [Transport Security 传输阶段加密](#transport-security-%e4%bc%a0%e8%be%93%e9%98%b6%e6%ae%b5%e5%8a%a0%e5%af%86)
  - [Authentication 认证阶段](#authentication-%e8%ae%a4%e8%af%81%e9%98%b6%e6%ae%b5)
  - [Authorization 授权阶段](#authorization-%e6%8e%88%e6%9d%83%e9%98%b6%e6%ae%b5)
  - [Admission Control 准入控制](#admission-control-%e5%87%86%e5%85%a5%e6%8e%a7%e5%88%b6)
  - [API Server Ports and IPs](#api-server-ports-and-ips)

# Controlling Access to the Kubernetes API

普通用户、Kubernetes中的服务在使用API时需要通过权限认证，当一个请求到达API时，其经过几个阶段，如下图所示：
![Alt Text](access-control-overview.svg)

## Transport Security 传输阶段加密

在Kubernetes集群中，一般而言API server运行在6443端口之上。API server中存储着认证证书，该证书一般是自签署的(self-signed)。证书信息存储在文件/etc/kubernetes/config文件中。请求信息发送至集群过程中，请求数据使用该证书加密传输

## Authentication 认证阶段

`Authentication modules`：认证模块

当TLS建立后，HTTP请求进入认证阶段，也即图中所示的step1。在集群创建的时候，可通过配置文件或配置脚本编辑API server，以运行一个或多个`Authenticator Modules`。

常见的`Authenticator Modules`包括：`Client Certicates`（客户证书），`Password`，`Plain Tokens`，`Bootstrap Tokens`以及`JWT Tokens`（用于service accounts），在实际使用时可指定一个或多个`Authenticator Modules`，认证时会逐个依次尝试直到其中一个`Authenticator Modules`成功。

**Note:** 认证阶段的输入为整个HTTP请求，但是通常情况下，认证阶段仅检测HTTP头或者客户证书。

请求认证结果有两种：

1. 请求认证失败，该请求被拒绝，返回HTTP状态码401；
2. 请求成功，用户被认证为某一特定的`username`，该`username`在后续操作中可以正常使用；<span style="border-bottom: 2px solid red; font-weight: bold">部分authenticators提供了用户之间的组关系描述，但其他authenticators并不提供。</span>

**Note:** <span style="border-bottom: 2px solid red; font-weight: bold">尽管Kubernetes使用`username`用于权限控制决策，但Kubernetes并无`user`对象，且并不存储usernames或其他关于users的信息</span>

## Authorization 授权阶段

`Authorization Models`：授权模块

当一个请求被认证为某一特定用户，该请求进入Authorizaiton授权阶段，也即图中的step2。

一个请求需要包括`username`，`requested action`，以及对象受此action的影响。当存在某一策略声明该用户拥有完成`requested action`的权限，该request授权成功。

一个例子，Bob拥有在namespace `projectCaribou`的pods读权限:
```json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "bob",
        "namespace": "projectCaribou",
        "resource": "pods",
        "readonly": true
    }
}
```
当他发起如下请求时，该请求授权成功：
```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "projectCaribou",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    }
  }
}
```
而当他发起在`projectCaribou`中写操作，如`create`或`update`操作时，请求授权失败，当他发起在`projectFish`中的读操作，该请求授权被拒绝。

**Note:** Kubernetes的授权阶段建议使用通用的REST属性与其他的组织范围内的权限控制系统，云提供者范围内的权限控制系统进行交互。<span style="border-bottom: 2px solid red; font-weight: bold">这是十分重要的，因为其他的权限控制系统可能与其他非Kubernetes API进行交互。</span>

Kubernetes支持多种`authorization modules`，常见有`ABAC Mode`, `RBAC Mode`以及`Webhook mode`。在创建Kubernetes集群时可通过配置文件或配置脚本指定使用的`authorization modules`。如果配置了多种`authorization modules`，Kubernetes会逐个检查。请求授权结果：

1. 任何一个`authorization modules`授权通过，该请求通过；
2. 所有`authorization modules`授权失败，该请求失败，HTTP状态码返回403；

## Admission Control 准入控制
`Admission Control Modules`：准入控制模块，用于修改或拒绝请求。即图中的step3。

`Admission Control Modules`除`authorizaiton modules`可用的属性外，`Admission Control Modules`还可以访问正在创建或更新的对象的内容。<span style="border-bottom: 2px solid red; font-weight: bold">其作用于正在创建、删除、更新、链接的对象，但不读取对象。</span>

可指定多个`admission controllers`，当请求到达时有几种情况：
1. 任意一个`admisson controllers`拒绝该请求，该请求被立即拒绝；
2. 所有`admisson controllers`通过该请求，该请求通过，并通过合法的路由将请求路由至对应的API对象；

除了拒绝对象外，`admission controllers`可以设置对象域较为复杂的默认值。

## API Server Ports and IPs
默认情况下，Kubernetes API Server运行在Http协议上，常运行端口有两个：
1. `Localhost Port`:
   + 被用来测试和启动集群，同时也被用于`master node`的其他组件与API之间的通信;
   + 无TLS加密;
   + 默认端口为8080，可通过`--insecure-port`参数更改;
   + 默认的IP为localhost，可通过`--insecure-bind-address`参数修改;
   + 发送至该端口的请求绕过`authentication modules`以及`authorization modules`;
   + 发送至该端口的请求由`admission control modules`管理;
   + 受需要访问的主机保护;
2. `Secure Port`:
   + 建议在任何可能的情况下使用;
   + 使用TLS，证书通过参数`--tls-cert-file`设置，密钥通过参数`--tls-private-key-file`设置;
   + 默认端口为6443，可通过`--secure-port`参数设置;
   + 默认IP为第一个非localhost网络的网络接口，可通过参数`--bind-address`更改;
   + 请求通过`authentication modules`及`authorization modules`;
   + 请求被`admission control modules`管理;
   + `authentication modules`及`authorization modules`运行
