- [Service](#service)
  - [Motivation](#motivation)
  - [Service resources](#service-resources)
  - [Defining a Service](#defining-a-service)

# Service

Kubernetes service是一个将运行在Pods集合中的应用导出为网络服务的抽象方法。使用Kubernetes时无需修改应用使用陌生的服务发现机制。Kubernetes给Pods以独立的IP地址，以及Pods集合的DNS名称，通过这种方式，使流量在Pods集合中负载均衡。

## Motivation

This leads to a problem: if some set of Pods (call them "backends") provides functionality to other Pods (call them "frontends") inside your cluster, how do the frontends find out and keep track of which IP address to connect to, so that the frontend can use the backend part of the workload?

Enter Services.

## Service resources

在Kubernetes中，一个Service是一个抽象概念，这个概念定义了一个Pods的逻辑集合以及访问该逻辑集合的相关规则。service通过selector定位后端Pods。

一个例子，考虑一个image-processing作为后端的stateless，该stateless运行3个replicas。这些副本是可替代的，前端无需关注其具体使用的后端。尽管在使用过程中后端发生了变换，前端使用时不会感知其变化。

## Defining a Service

Kubernetes中Service是一个REST对象。与所有REST对象类似，用户可通过使用`POST`将一个service定义推送至API server以创建一个新的instance。Service object的名称必须是一个合法的DNS label name。

一个例子，假定有一个Pods集合，集合中的每个Pod监听9376端口，并拥有一个`app=MyApp`的label。通过下列yaml文件可定义如下Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```


