- [Kubernetest Device Plugin](#kubernetest-device-plugin)
  - [Device plugin registration](#device-plugin-registration)
  - [Use The Resources](#use-the-resources)

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