- [Kubernetes API权限控制](#kubernetes-api%e6%9d%83%e9%99%90%e6%8e%a7%e5%88%b6)
  - [Transport Security 传输阶段加密](#transport-security-%e4%bc%a0%e8%be%93%e9%98%b6%e6%ae%b5%e5%8a%a0%e5%af%86)
  - [Authentication 认证阶段](#authentication-%e8%ae%a4%e8%af%81%e9%98%b6%e6%ae%b5)

# Kubernetes API权限控制
普通用户、Kubernetes中的服务在使用API时需要通过权限认证，当一个请求到达API时，其经过几个阶段，如下图所示：
![Alt Text](access-control-overview.svg)

## Transport Security 传输阶段加密
在Kubernetes集群中，一般而言API server运行在6443端口之上。API server中存储着认证证书，该证书一般是自签署的(self-signed)。证书信息存储在文件/etc/kubernetes/config文件中。请求信息发送至集群过程中，请求数据使用该证书加密传输

## Authentication 认证阶段
当TLS建立后，HTTP请求进入认证阶段，也即图中所示的step1。在集群创建的时候，可通过配置文件或配置脚本编辑API server，以运行一个或多个`Authenticator Modules`。

常见的`Authenticator Modules`包括：`Client Certicates`（客户证书），`Password`，`Plain Tokens`，`Bootstrap Tokens`以及`JWT Tokens`（用于service accounts），在实际使用时可指定一个或多个`Authenticator Modules`，认证时会逐个依次尝试直到其中一个`Authenticator Modules`成功。

**Note:** 认证阶段的输入为整个HTTP请求，但是通常情况下，认证阶段仅检测HTTP头或者客户证书。

请求认证结果有两种：
1. 请求认证失败，该请求被拒绝，返回HTTP状态码401；
2. 请求成功，用户被认证为某一特定的`username`，该`username`在后续操作中可以正常使用；<span style="border-bottom: 2px solid red; font-weight: bold">部分authenticators提供了用户之间的组关系描述，但其他authenticators并不提供。</span>

**Note:** **尽管Kubernetes使用`username`用于权限控制决策，但Kubernetes并无`user`对象，且并不存储usernames或其他关于users的信息**
