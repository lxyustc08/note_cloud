- [DNS for Services and Pods](#dns-for-services-and-pods)
  - [Introduction](#introduction)
    - [Namespaces of Services](#namespaces-of-services)
    - [DNS Records](#dns-records)
  - [Services](#services)
    - [A/AAAA records](#aaaaa-records)
    - [SRV records](#srv-records)
  - [Pods](#pods)
    - [A/AAAA records](#aaaaa-records-1)
    - [Pod's hostname and subdomain fields](#pods-hostname-and-subdomain-fields)
    - [Pod's setHostnameAsFQDN field](#pods-sethostnameasfqdn-field)
    - [Pod's DNS Config](#pods-dns-config)

# DNS for Services and Pods

Kubernetes为services以及pods创建DNS记录，用户通过使用持久化的DNS名称与服务连接。

## Introduction

Kubernetes DNS在集群上调度DNS Pod和服务，并配置kubelet告知各容器使用DNS服务的IP来解析DNS名称

每个集群中定义的服务（包括DNS server本身）被赋值一个DNS name。默认情况下，一个客户端Pod的DNS搜索列表包括Pod本身的namespace以及集群的默认域。

### Namespaces of Services

在Pod发出DNS查询请求时，根据Pod namespace的不同，DNS查询请求返回结果存在区别。在Kubernetes中，DNS查询时未指定namespace时，该DNS查询仅限于发出查询请求的Pod所在的namespace。当要对其他namespace中的service发起DNS查询请求时，需要指定该DNS查询请求所发生的namespace。

一个例子，名称为test的namespace中存在一个pod，名称为prod的namespace中存在名称为data的service，现在在名称为test的namespace的pod中发起DNS查询请求：

+ 对于data的DNS查询，返回空结果，此时使用的是pod的test namespace，该namespace中无data service存在
+ 对于data.prod的DNS查询，返回预期的结果，此时DNS查询请求指定了prod namespace

DNS查询时，Kubernetes会根据Pod's中的文件`/etc/resolv.conf`对DNS查询对象进行扩展，文件`/etc/resolv.conf`由kubelet进行设置。扩展时，对于上述例子中的data而言，其将扩展为`data.test.cluster.local`，文件/etc/resolv.conf中的search选项，对扩展方式进行了配置。

### DNS Records

下述对象将拥有DNS records：

+ Services
+ Pods

每类对象支持的DNS records的类型机支持的布局后续内容中进行详细描述。

## Services

### A/AAAA records

非headless services被根据IP协议簇的不同，被赋值一个A（IPv4协议簇）或者AAAA类DNS记录（IPv6协议簇）；对于形式为`my-svc.my-namespace.svc.cluster-domain.example`而言，此时将被解析为该Service的ClusterIP

对于headless services而言，其同样赋值为一个A（IPv4协议簇）或者AAAA（IPv6协议簇）类DNS记录。但与非headless services不同，DNS解析时，解析为其通过selector形成的Pod的集合中的Pod IP，一个例子

```
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
cassandra   ClusterIP   None            <none>        9042/TCP            5d22h
nginx       ClusterIP   None            <none>        80/TCP              6d21h
```

通过使用busybox镜像中的nslookup命令对nginx进行DNS查询

```
nslookup nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.88.5.95 web-0.nginx.k8s-samples.svc.cluster.local
Address 2: 10.88.5.88 web-3.nginx.k8s-samples.svc.cluster.local
Address 3: 10.88.5.94 web-2.nginx.k8s-samples.svc.cluster.local
Address 4: 10.88.5.96 web-1.nginx.k8s-samples.svc.cluster.local
```

### SRV records

SRV records通常为创建用于headless service中的已命名端口，对于每个已命名的端口而言，SRV records具有以下形式`_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example`。

对于非headless service而言，SRV records解析时将解析到对应的端口号以及域名：`my-svc.my-namespace.svc.cluster-domain.example`。

对于headless service而言，SRV records解析时将解析到多个值，每个值为service的后端pod对应解析，包括端口号以及pod的域名`auto-generated-name.my-svc.my-namespace.svc.cluster-domain.example`

## Pods

### A/AAAA records

通常情况下，Pod具有如下的DNS地址，`pod-ip-address.my-namespace.pod.cluster-domain.example`

一个例子

```
NAME          READY   STATUS    RESTARTS   AGE     IP            NODE                         NOMINATED NODE   READINESS GATES
dns-test      1/1     Running   0          3s      10.88.6.149   worker-amd64-gpuceph-node3   <none>           <none>
```

使用busybox镜像中的nslookup命令查询`10-88-6-149.k8s-samples.pod.cluster.local`

```
nslookup 10-88-6-149.k8s-samples.pod.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      10-88-6-149.k8s-samples.pod.cluster.local
Address 1: 10.88.6.149 dns-test
```

通过Deployment或者DaemonSet创建，并通过Service进行导出的Pods具有下列形式的DNS解析形式`pod-ip-address.deployment-name.my-namespace.svc.cluster-domain.example`

### Pod's hostname and subdomain fields

当Pod被创建时，其hostname默认为Pod的`metadata.name`域的值。

Pod spec拥有一个可选的`hostname`域，该域可被用于指定Pod的hostname。当`hostname`域被指定后，其值将覆盖`metadata.name`域的值。

Pod spec拥有一个可选的`subdomain`域，该域被用来指定Pod的subdomain。设置该域后，Pod的默认DNS记录规则被改编为FQDN名称

一个例子：一个Pod的`hostname`域被设置为`"foo"`，其`subdomain`域被设置为`"bar"`，该pod处于`"my-namespace"`中，则此时该Pod的FQDN为"`foo.bar.my-namespace.svc.cluster-domain.example`"

若同一个namespace中存在与Pod的subdomain一致的名称的headless service，集群的DNS server同样返回一个A或者AAAA类记录。一个例子：Pod的`hostname`域被设置为`"busybox-1"`，subdomain被设置为"`default-subdomain`"；同时在同一个namespace中存在一个名称为"`default-subdoamin`"的headless service，此时，Pod的FQDN为"`busybox-1.default-subdomain.my-namespace.svc.cluster-domain.example`"；集群的DNS将产生一个名称为上述FQDN，地址为指向Pod IP地址的A类DNS记录。"`busybox1`"以及"`busybox2`"拥有不同的A或者AAAA记录。

### Pod's setHostnameAsFQDN field

当Pod被配置为使用FQDN时，他的hostname即为short hostname。一个例子，若一个Pod的FQDN为`busybox-1.default-subdomain.my-namespace.svc.cluster-domain.example`，命令`hostname -f`返回值为busybox-1。

当设置setHostnameAsFQDN为ture时，kubelet将Pod's的FQDN写入该Pod命名空间的hostname。在这种情况下，hostname以及hostname --f均返回Pod的FQDN

> Note: In Linux, the hostname field of the kernel (the nodename field of struct utsname) is limited to 64 characters.

> If a Pod enables this feature and its FQDN is longer than 64 character, it will fail to start. The Pod will remain in Pending status (ContainerCreating as seen by kubectl) generating error events, such as Failed to construct FQDN from pod hostname and cluster domain, FQDN long-FQDN is too long (64 characters is the max, 70 characters requested). One way of improving user experience for this scenario is to create an admission webhook controller to control FQDN size when users create top level objects, for example, Deployment.

### Pod's DNS Config

Pod's的DNS Config允许用户对Pod的DNS设置进行更为精细的控制。DNS config牵涉到2个配置域：`dnsConfig`以及`dnsPolicy`，其中`dnsConfig`是可选域，可以任何`dnsPolicy`配置进行协作。但是，一旦当Pod's的`dnsPolicy`被设置为"`None`"时，`dnsConfig`必须被设置。域`dnsConfig`用户可配置的属性如下：

+ `nameservers`：a list of IP addresses that will be used as DNS servers for the Pod. There can be at most 3 IP addresses specified. When the Pod's dnsPolicy is set to "None", the list must contain at least one IP address, otherwise this property is optional. The servers listed will be combined to the base nameservers generated from the specified DNS policy with duplicate addresses removed.
+ `searches`：a list of DNS search domains for hostname lookup in the Pod. This property is optional. When specified, the provided list will be merged into the base search domain names generated from the chosen DNS policy. Duplicate domain names are removed. Kubernetes allows for at most 6 search domains.
+ `options`：an optional list of objects where each object may have a name property (required) and a value property (optional). The contents in this property will be merged to the options generated from the specified DNS policy. Duplicate entries are removed.

一个使用自定义DNS setting的Pod配置：

```yaml
apiVersion: v1
kind: Pod
metadata:
        namespace: k8s-samples
        name: dns-example
spec:
        containers:
                - name: test
                  image: lxyustc.registrydomain.com:5000/nginx:alpine
        dnsPolicy: "None"
        dnsConfig:
                nameservers:
                        - 1.2.3.4
                searches:
                        - ns1.svc.cluster-domain.example
                        - my.dns.search.suffix
                options:
                        - name: ndots
                          value: "2"
                        - name: edns0
```

部署上述配置后，执行如下命令

```
kubectl exec -it dns-example -n k8s-samples -- cat /etc/resolv.conf

search ns1.svc.cluster-domain.example my.dns.search.suffix
nameserver 1.2.3.4
options ndots:2 edns0
```
