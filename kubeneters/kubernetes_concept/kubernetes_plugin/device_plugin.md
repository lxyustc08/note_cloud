- [Kubernetest Device Plugin](#kubernetest-device-plugin)
  - [Device plugin registration](#device-plugin-registration)
  - [Use The Resources](#use-the-resources)
  - [Device plugin implementation](#device-plugin-implementation)
    - [Handling kubelet restarts](#handling-kubelet-restarts)
  - [Device plugin deployment](#device-plugin-deployment)
  - [API compatibility](#api-compatibility)
  - [Monitoring Device Plugin Resources](#monitoring-device-plugin-resources)
    - [Monitoring agents Deployment](#monitoring-agents-deployment)
  - [Device plugin integration with the Topology Manager](#device-plugin-integration-with-the-topology-manager)

# Kubernetest Device Plugin

Kubernetes从1.10开始提供Device Plugin机制，该机制设计了一种`device plugin framework`，利用该框架可以实现供GPUs、NICs、FPGAs、InfiniBand等类似的需要由设备厂商配置的硬件资源设置。

利用`device plugin framework`，用户可将系统具备的硬件资源告知`kubelet`。

通过使用`device plugin framework`，设备厂商无需定制`kubelet`代码，用户方可通过Kubernetes的`DaemonSet`或者手动部署插件的方式实现硬件的利用。

## Device plugin registration

kubelet导出一个名称为`Registration`的gPRC服务

```golang
service Registration {
  rpc Register(RegisterRequest) returns (Empty) {}
}
```

通过上述gPRC服务device plugin向`kubelet`注册自己，在注册的过程中device plugin需要提供如下信息：

+ The name of its Unix Socket
+ The Device plugin API version against whitch is was built
+ The `ResourceName` it wants to advertise. Here `ResourceName` needs to follow the extended resource nameing scheme as vendor-domain/resourcetype. (for example, nvidia GPU is nvidia.com/gpu)

## Use The Resources

当device plugin注册后，用户可在container specification中请求其所需的资源类型，使用过程中需要注意如下**限制**：

+ **Extended resources are only supported as integer resources an cann't be overcommitted**
+ **Device cannt be shared ammong containers**

一个典型例子：

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: demo-container-1
      image: k8s.gcr.io/pause:2.0
      resources:
        limits:
          hardware-vendor.example/foo: 2
```

## Device plugin implementation

+ Initialization, 在这一阶段，device plugin执行制造商规定的初始化与设置流程，确保device处于ready状态
+ 启动gPRC服务，并在host上的`/var/lib/kubelet/device-plugins/`路径下创建一个Unix socket，gPRC服务实现如下的接口：
  ```golang
    service DevicePlugin {
        // ListAndWatch returns a stream of List of Devices
        // Whenever a Device state change or a Device disappears, ListAndWatch
        // returns the new list
        rpc ListAndWatch(Empty) returns (stream ListAndWatchResponse) {}

        // Allocate is called during container creation so that the Device
        // Plugin can run device specific operations and instruct Kubelet
        // of the steps to make the Device available in the container
        rpc Allocate(AllocateRequest) returns (AllocateResponse) {}
  }
  ```
+ plugin注册自己，通过`/var/lib/kubelet/device-plugins/kubelet.sock`注册
+ 以`serve mode`模式运行，在这个过程中，持续对device health进行检测，并向`kubelet`报告一切device state的状态变化，同时负责处理`Allocate`gPRC请求。在`Allocate`过程中，`device plugin`可能处理device-specific相关的准备工作。比如GPU cleanup或者QRNG初始化。如果相关操作成功执行，`device plugin`返回一个`AllocateResponse`，该响应包含所需访问已分配device的container runtime配置，`kubelet`将配置信息传输至container runtime

### Handling kubelet restarts

对于device plugin而言，其需要实现`kubelet`重启探测，当kubelet重启后，device plugin需要重新向`kubelet`注册自己。

**当前`kubelet`实现是，当kubelet重启时，新的kubelet实例将删除文件夹/var/lib/kubelet/device-plugins下所有内容。这样的话对于device plugin而言，可以探测到其上次注册时生成的socket被删除，然后可以根据这一事件重新注册自己。**

## Device plugin deployment

可通过DaemonSet方式部署device plugin，当然也可以通过手动逐个节点部署device plugin。

采用DaemonSet方式部署device plugin，需要将/var/lib/kubelet/device-plugins作为Pod spec中的volume。一个典型的Nvidia GPU k8s插件：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      # This annotation is deprecated. Kept here for backward compatibility
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      # This toleration is deprecated. Kept here for backward compatibility
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      - key: CriticalAddonsOnly
        operator: Exists
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      # Mark this pod as a critical add-on; when enabled, the critical add-on
      # scheduler reserves resources for critical add-on pods so that they can
      # be rescheduled after a failure.
      # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
      priorityClassName: "system-node-critical"
      containers:
      - image: lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6
        name: nvidia-device-plugin-ctr
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
```

通过DaemonSet方式部署device plugin可以较好利用kubernetes特性，比如失效处理，集群节点升级等。

## API compatibility

Kubernetes device plugin的特性仍处于beta阶段，相关的API仍处于变化过程中，需要十分注意相关变化

## Monitoring Device Plugin Resources

为了监控由`device plugin`提供的resources，monitoring agent需做两方面的工作：

+ discover the device that are in-use on the node
+ obtain the metadata to describe which container the metric should be associated with

**在metric维护方面**，device monitoring 导出的[Prometheus](https://prometheus.io/) metrics需符合[Kubernetes Instrumentation Guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/instrumentation.md)，并通过Prometheus的`pod`, `namespace`以及`container`标签对container进行区分

**在device发现方面**，kubelet提供了如下的gRPC服务供设备发现使用：

```golang
// PodResourcesLister is a service provided by the kubelet that provides information about the
// node resources consumed by pods and containers on the node
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
}
```

上述gRPC服务同样通过Unix socket `/var/lib/kubelet/pod-resources/kubelet.sock`提供。

### Monitoring agents Deployment

Monitoring agents同样可通过DaemonSet或手动部署daemon的方式进行部署。由于目录/var/lib/kubelet/pod-resouces需要特权访问，因此monitoring agents需要运行在特权安全的上下文中。

如果通过DaemonSet的方式部署monitoring agent，/var/lib/kubelet/pod-resources必须指定为插件Podsepc中的Volume。

对于PodResrouces service的监控，需要开启`KubeletPodResources`这一特性([feature gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/))，从1.15版本开始，该特性默认开启

<!-- TODO Topology机制需要研究理解 -->

## Device plugin integration with the Topology Manager

The Topology Manager is a Kubelet component that allows resources to be co-ordintated in a Topology aligned manner. In order to do this, the Device Plugin API was extended to include a TopologyInfo struct.

```golang
message TopologyInfo {
	repeated NUMANode nodes = 1;
}

message NUMANode {
    int64 ID = 1;
}
```

Device Plugins that wish to leverage the Topology Manager can send back a populated TopologyInfo struct as part of the device registration, along with the device IDs and the health of the device. The device manager will then use this information to consult with the Topology Manager and make resource assignment decisions.

`TopologyInfo` supports a `nodes` field that is either nil (the default) or a list of NUMA `nodes`. This lets the Device Plugin publish that can span NUMA nodes.
