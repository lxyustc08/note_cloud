- [Istio Traffic Management](#istio-traffic-management)
  - [Istio 流量管理介绍](#istio-流量管理介绍)
  - [Virtual services](#virtual-services)
    - [使用virtual services的目的](#使用virtual-services的目的)
    - [Virtual service 例子](#virtual-service-例子)
      - [Host field](#host-field)
      - [Routing rules](#routing-rules)
        - [Match condition](#match-condition)
        - [Destination](#destination)
      - [Routing rule precedence](#routing-rule-precedence)
      - [More about routing rules](#more-about-routing-rules)
  - [Destination rules](#destination-rules)
    - [Load balancing options](#load-balancing-options)
    - [Destination rule example](#destination-rule-example)
  - [Gateways](#gateways)
    - [Gateway example](#gateway-example)
  - [Service entries](#service-entries)
    - [Service entry example](#service-entry-example)
  - [Sidecars](#sidecars)
  - [Network resilience and testing](#network-resilience-and-testing)
    - [Timeouts](#timeouts)
    - [Retries](#retries)
    - [Circuit breaks](#circuit-breaks)
    - [Fault injection](#fault-injection)

# Istio Traffic Management

Istio流量管理用于控制services之间的流量流向以及API调用。

Istio将诸如circuit breakers, timeouts, and retries配置进行简化，同时简化了诸如A/B tesing，canary rollouts, and staged rollouts等重要任务的配置，可实现精确到百分比的流量划分。在可靠性方面，他提供了开箱即用的故障恢复功能，帮助用户应用在失效上有更强的鲁棒性。

## Istio 流量管理介绍

Istio进行流量管理时需要获取两方面的信息：

+ 服务endpoints入口位置
+ endpoints归属的服务名称

Istio维护了一个Service Registry以存储上述两方面的信息，Istio本身并不提供服务发现功能，而是通过连接服务发现系统实现Service Registry的内容填充。

> 虽然通过Pilot可实现大部分服务的自动化注册，但本质上Pilot仅反应了底层平台的服务发现机制。

通过使用Service Registry，Envoy可将流量引导至相关的服务中去。大部分微服务化的应用拥有多个用于管理流量的实例instances，这些实例有时候被称为负载均衡池。默认情况下，Envoy使用轮询模式处理负载均衡池中的流量。

尽管Istio提供了基本的服务发现与负载均衡机制，但Istio针对更精细化的流量管理，提供更为高阶的流量管控功能。这些功能通过Kubernetes提供custom resource definition (CRDs)机制进行管理。

由Istio提供的用于进行流量管理的资源如下：

+ [Virtual services](#virtual-services)
+ Destination rules
+ Gateways
+ Service entries
+ Sidecars

## Virtual services

Virtual services时Istio流量管理的核心构建块。一个virtual service在Istio以及底层平台提供基本的连接性与服务发现的基础上，允许用户对请求如何路由到istio内部的服务网格的方式进行配置。

每个virtual service是一个包含路由规则的集合，这些路由规则按照顺序进行计算。通过这种方式允许Istio将传递给virtual service的请求传递到真实的目的地址。根据需求，用户可配置多个virtual services

### 使用virtual services的目的

virtual services在Istio的流量管理过程中扮演了重要的角色，通过virtul services可实现Istio流量管理的功能的高弹性及功能上的强力性。

通过virtual services，可将客户端发送请求与实现请求的最终目的负载解耦。

> + Without virtual services, Envoy distributes traffic using round-robin load balancing between all service instances, as described in the introduction.
> + With a virtual service, you can specify traffic behavior for one or more hostnames.
>   + You use routing rules in the virtual service that tell Envoy how to send the virtual service’s traffic to appropriate destinations. Route destinations can be versions of the same service or entirely different services.

**一个典型的使用场景：将流量分发至不同版本的服务。**

+ 对于客户端而言，其发送请求至virtual service hosts，将发送的virtual service hosts视为一个单独的entity，`a single entity`
+ Envoy将按照配置将客户端发送至virtual service hosts的流量进行转发，比如：
  + 20%的流量分发至新版本
  + 特定范围内的用户流量分发至版本2

通过上述机制，流量的路由过程完全与服务实例的部署分隔，这也意味着，service instances的个数可根据实际的流量负载scale up/ scale down

除上述典型场景外，virtual services给予用户如下实现能力

+ Address multiple application services through a single virtual service
+ Configure traffic rules in combination with gateways to control ingress and egress traffic

当然，除了需要配置virtual service外，还需要配置destination rules，在Istio定义的CRD中，destination rules与virtual service互相独立配置，采用不同的配置对象。

### Virtual service 例子

一个典型的vritual service例子如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

#### Host field

`host`域列出了virtual service的主机名称，换而言之，是用户可寻址的目的以及route规则生效的目的。该地址，也是用户发送请求时的地址。

> 对于客户端/用户而言，其只可看到virtual service

virutal service hostname可以是IP地址，DNS名称，或者平台相关的名称（如Kubernetes的资源缩写名称）。

用户可以使用通配符("*")前缀，这种方式允许用户创建一个单独的路由规则集合，该集合对于所有的匹配服务生效。

Virtual service host并不需要作为Istio的服务仓库的一部分，他们仅仅是简单的虚拟目的地址。这可以允许用户对mesh中的不可路由条目进行流量建模。

#### Routing rules

```yaml
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

例子中的http section部分即为virtual service的路由规则。

路由规则支持对HTTP/1.1, HTTP2, and gRPC进行流量配置。

每条路由规则由两部分组成：

+ the destiantion where you want the traffic to go
+ zero or more match conditions

##### Match condition

第一条路由规则

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

该条路由规则含义如下：匹配所有来源于用户jason的请求。

##### Destination

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

`route` section中的`destination`域指定了符合match condition的流量的目的地址。

不同于virtual service name，destination处的host必须为Istio service registry真实存在的。该处的host既可以是一个mesh service也可以是一个通过service entry增加的非mesh service。

上述例子中的destination hosts采用的是Kubernetes中的short name。在规则被计算时，istio基于namespace添加domain前缀，从而获取到host完整的qualified name。

> 上述使用short name的方式要求destination hosts与virtual service位于同一个Kubernetes namespace中。若位于不同的namespace中将导致配置丢失，因此在生产环境下，建议指定完整的qualified host names

#### Routing rule precedence

routing rule的计算顺序从顶到下，virtual service定义中的第一条规则优先度最高。

借鉴于上述机制，可在规则中设置一条默认规则，此规则在优先级高的规则不生效时生效，比如例子中的最后一条规则

```yaml
- route:
  - destination:
      host: reviews
      subset: v3
```

通常情况下，默认规则无需设置condition或者设置为基于权限的规则。

#### More about routing rules

关于路由规则的进一步讨论

在匹配规则的设置上，可设置多种类型的条件，比如流量端口，header field，URIs等。如下面一个例子：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
```

在设置匹配条件时，还可选择exact value, prefix或者regex。

在设置匹配条件时还可对同个匹配块设置多个条件，实现匹配条件的逻辑与；也可将多个匹配条件设置为同个匹配规则，实现逻辑或。

除了使用匹配条件，用户可以使用"weight"，该值以百分比对流量进行控制。

```yaml
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
```

用户可以使用路由规则对流量执行相关操作，比如：

+ Append or remove headers
+ Rewrite the URL
+ Set a retry policy for calls to this desination

## Destination rules

跟virtual services类似，`destination rules`也是Istio实现流量控制功能的关键部分。

Destination rules与Virtual services的区别

+ Virtual services
  + 用户配置流量如何路由到给定的destination
+ Destination rules
  + 用户通过destination rules对抵达目的地的流量进行配置

Destination rules在Virtual services的route规则计算后再被计算，因此destination rules应用的对象是流量的真实目的地址（traffic's "real" destination）

特别地，用户可使用destination rules来指定命名的服务子集，比如按照版本对所有给定服务的实例进行分组。通过这种方式，用户可在virtual services中的routing rules使用这些服务子集。

Destination rules同样允许用户在调用整个destination service或者一个特定的service子集时对Envuy的流量规则进行配置。比如选择的负载均衡模式，TLS加密模式以及断路器设置等。

### Load balancing options

默认情况下Istio使用轮询作为默认的负载均衡策略，除轮询外，Istio同样支持如下模式：

+ Random
  + Requests are forwarded at random to instances in the pool
+ Weighted
  + Requests are forwarded to instances in the pool according to a specific percentage.
+ Least requests
  + Requests are forwarded to instances with the least number of requests

### Destination rule example

一个Destination rule的例子

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

每个子集通过一个或多个label进行定义。这些lables在Kubenretes的服务部署中作为Kubernetes服务的`metadata`域中的值对不同的版本进行标识。

上述例子中，除了对subsets进行定义外，destination rule定义了默认的流量规则，即出现在subsets上的trafficPolicy，该默认规则可被子集进行覆盖。

## Gateways

用户使用gateway管理进出mesh的inbound与outbound流量，指定哪些流量允许进出mesh。

从Envoy角度来看，Gateway对运行在mesh边缘的标准Envoy代理进行编辑，而非对与服务负载同时运行的Envoy代理进行编辑。

不同于Kubernetes Ingress APIs等其他的控制流量进出的机制，Istio的gateway允许用户充分利用Istio流量路由的弹性和强功能性。Istio Gateway资源仅允许用户编辑4-6层负载均衡属性，比如端口暴露规则，TLS设置等。若要设置7层流量路由设置，用户可将一个Istio virtual service资源绑定到gateway上，这种特性可使用户想管理其他data plane流量一样管理gateway traffic。

gateway通常被使用来管理入口流量，但是用户同样可以使用gateway来编辑egress gateways。通常一个egress gateway被用来配置哪个节点作为mesh流量的退出节点，也可以被配置为哪些服务可以从外部网络访问，也可以配置为安全控制下的egress traffic，来增加mesh的security

Istio提供一些预先配置的gateway proxy deployments(`istio-ingressgateay`以及`istio-egressgateway`)，这些gateway proxy deployments在用户使用demo installation配置时会进行部署，若使用default profile进行Istio部署则仅需会部署`istio-ingressgateway`

### Gateway example

一个典型的Istio gateway如下所示

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      credentialName: ext-host-cert
```

上述例子允许HTTPS流量从ext-host.example.com的443端口进入mesh。如前所述gateway并未指定流量的路由规则，要指定gateway的路由的规则，需将gateway绑定至一个virtual service上，如下所示

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
  - ext-host-gwy
```

## Service entries

用户可以使用service entries将service entry添加至由Istio维护的服务仓库中。当service entry被添加至服务仓库后，Envoy proxies可发送流量至该service entry，如同mesh中已运行的service一样。

通过配置service entries，用户可以对运行在mesh外的服务流量进行如下管理：

+ Redirect and forward traffic for external destination, such as APIs consumed from the web, or traffic to services in legacy infrastructure.
+ Define retry, timeout, and fault injection policies for external destinations
+ Run a mesh service in a Virtual Matchine by adding VMs to your mesh
+ Logically add services from a different cluster to the mesh to configure a multicluster Istio mesh on Kubernetes

### Service entry example

一个典型的服务目录例子如下

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

上述例子将名称为`ext-svc.example.com`的外部服务依赖添加至Istio的服务仓库中。

用户可以配置virtual services以及destination rules来精细化控制service entry的流量，如同mesh中的服务一样，如下一个例子，对名称为`ext-svc.example.com`的外部服务配置destination rule，该destination rule使用TLS来加密`ext-svc.example.com`的连接：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

## Sidecars

默认情况下，Istio对每个Envoy proxy进行配置，以使Envoy接受其关联的负载的接口上的所有流量，并在转发流量时达到mesh中的每个工作负载。通过使用sidecar，可进行如下配置：

+ Fine-tune the set of ports and protocols that an Envoy proxy accepts
+ Limit the set of services that the Envoy proxy can reach

在实际的大型应用中，用户可限制Sidecar的可达性，以保障mesh的性能。

用户可将某一sidecar配置应用到特定namespace的所有负载中，或者通过`workloadSelector`域配置特定的负载。

一个典型例子如下所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

该例子的sidecar配置将`bookinfo` namespace中的所有服务可达性配置为同namespace或者Istio control plane。

## Network resilience and testing

Istio提供了一些可靠性与容错特性帮助用户应用的运行地更为可靠，典型特性如下：

### Timeouts

Timeout配置地是一个Envoy proxy在获取特定地service时应该等待的时长，通过timeout保证服务不会处于永不休止的等待状态，并确保调用在一个可预测的时间帧成功或失败。对于HTTP请求而言，Envoy的timeout配置默认禁止。

Istio允许用户方便地对每个服务的timeout进行配置，仅需在virtual services中编辑timeout域即可

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

### Retries

Retry setting指定了最大的重试次数，与Timeout类似，用户可以较为方便的配置每个服务的retry，仅需在特定的Virtual services中进行配置即可，如下例子所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

### Circuit breaks

通过circuit breaks，用户可对service中的每个hosts进行limits配置，当limits达到circuit breaks的设置值时，将主动中断后续的请求。（防止雪崩效应）

用户通过配置destination rules来配置circuit breaks，一个典型的例子如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
```

该例子中对名称为reviews的服务的v1版本的最大连接数配置为100。

### Fault injection

故障注入，Istio提供了故障注入特性，帮助用户对应用的故障恢复特性设计进行测试。

Istio的Fault injection允许对应用层进行故障注入。Istio允许注入两类故障，这两类故障均通过virtual service进行配置：

+ Delays:  Delays are timing failures. They mimic increased network latency or an overloaded upstream service.
+ Aborts: Aborts are crash failures. They mimic failures in upstream services. Aborts usually manifest in the form of HTTP error codes or TCP connection failures.

一个典型的故障注入例子：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

该配置对名称为ratings的virtual service，每1000个请求引入一个5s延迟
