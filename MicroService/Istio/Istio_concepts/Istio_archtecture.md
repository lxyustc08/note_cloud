- [Istio Archtecture](#istio-archtecture)
  - [Components](#components)
    - [Envoy](#envoy)
    - [Istiod](#istiod)

# Istio Archtecture

Istio 架构图如下所示

![Alt Text](../Istio_picture/arch.svg "Istio架构图")

## Components

### Envoy

Istio使用一个扩展的Envoy代理，Envoy是一个由C++编写的高性能代理软件，用于服务网格中服务流量的进出管理。Envoy代理是唯一一个同数据平面交互的组件。

Istio中Envoy代理以Sidecar方式部署，扩充了服务如下特性：

+ Dynamic service discovery
+ Load balancing
+ TLS termination
+ HTTP/2 and gRPC proxies
+ Circuit breakers
+ Health checks
+ Staged rollouts with %-based traffic split
+ Fault injection
+ Rich metrics

由Envoy proxies实现的Istio特性包括如下内容：

+ Traffic control features: enforce fine-grained traffic control with rich routing rules for HTTP, gRPC, WebSocket, and TCP traffic
+ Network resiliency features: setup retries, failovers, circuit breakers, and fault injection.
+ Security and authentication features: enforce security policies and enforce access control and rate limiting defined through the configuration API.
+ Pluggable extensions model based on WebAssembly that allows for custom policy enforcement and telemetry generation for mesh traffic

### Istiod

Istiod提供了服务发现、服务配置以及认证管理功能。

Istiod将高层的路由规则转换为Envoy配置，并将规则推送至处于运行状态的sidecars中

Pilot将平台相关的服务发现机制进行抽象并将其合成为一个标准的格式，保证任何符合Envoy API的sidecar均可识别
