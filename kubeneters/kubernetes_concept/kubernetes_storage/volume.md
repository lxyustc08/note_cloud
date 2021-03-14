- [Kubernetes Volume](#kubernetes-volume)
  - [Concept of Volume](#concept-of-volume)
  - [Types of Volume](#types-of-volume)
  - [Using subPath](#using-subpath)
    - [Using subPath with expanded environment variables](#using-subpath-with-expanded-environment-variables)
  - [Resources](#resources)
  - [Out-of-tree volume plugins](#out-of-tree-volume-plugins)
    - [CSI](#csi)
      - [CSI raw block volume support](#csi-raw-block-volume-support)
      - [CSI ephemeral volumes](#csi-ephemeral-volumes)
      - [Migrating to CSI drivers from in-tree plugins](#migrating-to-csi-drivers-from-in-tree-plugins)
    - [flexVolume](#flexvolume)
  - [Mount propagation](#mount-propagation)

# Kubernetes Volume

## Concept of Volume

Docker中也有Volume的概念，一般为本地文件夹或者容器卷，在功能上存在一定的受限，Kubernetes对于Volume进行了类似于虚拟机的概念扩展。

Kubernetes具有多种类型的Volume，Kubernetes的Pods可同时使用多种类型的Volume。

总体而言，Kubernetes的Volume可分为两种类型——持久化的Volume以及非持久化的Volume。

+ 非持久化的Volume生命周期与Pod一致，Pod摧毁其生命终止
  + 非持久化的Volume生命周期独立于Pod中的container，在container终止后其数据仍然存在
+ 持久化的Volume生命周期独立于Pod而存在

## Types of Volume

Volume本质上即为一个存储数据可被Pod中的Container访问的文件夹，该文件夹如何创建，如何被备份以及文件夹中的内容表现形式取决于Volume的类型

Kubernetes的Volume类型有如下几种：

+ awsElasticBlockStore
  + AWS提供的弹性存储卷EBS volume
+ azureDisk
  + 微软提供的Azure Data Disk
+ cephfs
  + 分布式存储ceph的cephfs
+ cinder
  + IaaS软件OpenStack提供的cinder卷存储
+ configMap
  + configMiap用于向pods传递所需配置信息，具体参见此部分[ConfigMap](../kubernetes_configuration/kubernetes_config_maps.md)
+ downwardAPI
  + downwardAPI提供了应用所需的可供下载的API数据，待研究使用
+ emptyDir
  + emptyDir在Pod被调度到node时即被创建，跟Pod生命周期一致，并跟Pod同时运行在被调度的node节点上。
  + emptyDir初始状态为空，所有Pod中的container可以读写empty中的文件。
  + emptyDir被Pod中的容器挂载时，各容器可指定不同的挂载路径
  + 当Pod被调度终止，emptyDir中的所有数据永久删除
  + emtpyDir用途如下
    + 暂存空间，用于注入基于磁盘归并排序
    + 检查点，用于备份长计算过程中的数据
    + 存储内容容器提供给web容器所需对外服务的数据
  + emptyDir存储在节点上的媒体文件中，包括如下媒体文件
    + 节点上的磁盘（HDD SSD）
    + 节点使用的网络存储
    + 内存文件，即将empty.medium设置为memory
      + 此时kubernetes将挂载一个tmpfs文件
+ fc
  + fc设备提供的光纤存储
+ gcePersistentDisk
  + 由Google GCE提供持久化存储卷persistent disk
+ glusterfs
  + 由Glusterfs（开源网络文件存储）提供的volume
+ hostPath
  + hostPath将host节点的文件或文件夹挂载至Pod中
  + hostPath为应用提供了强力的escape hatch（逃逸窗口），通常用途如下
    + 运行的容器需要访问内部的Docker，将hostPath设置为`/var/lib/docker`
    + 需要在容器中运行cAdvisor，hostPath此时为/sys
    + 允许Pod指定给定的hostPath是否需要在Pod运行前存在，是否需要被创建，以何种形式存在
    + 在使用hostPath时需注意如下两点
      + 具备完全相同的配置的Pods，在不同节点上由于其文件行为不同，其行为也存在不同
      + 若host的文件或文件夹仅能被root访问，在使用hostPath时，需借助特权容器或者将host的文件属性修改，以可写入hostPath
+ iscsi
  + iscsi设备提供的iSCSI卷
+ local
  + local volume即为已挂载的存储设备，诸如disk，分区或者文件夹
  + local volume仅可被使用为静态的PersistentVolume，不支持Dynamic provisioning
  + local volume使用时需注意以下几点
    + 使用时需指定PersisitentVolume的节点亲和性
    + 使用时需考虑数据丢失的容错性。
+ nfs
  + 提供NFS网络文件系统
+ persistentVolumeClaim
  + persistentVolumeClaim被用于向Pod中挂载PersistentVolume
  + persistentVolumeClaim向用户提供了一种声明持久化存储的方式，用户无需了解特定的cloud environment细节
+ portworxVolume
  + 由Portworx提供的弹性块存储
+ projected
  + projected volume将现存的volume sources打包整合至一个文件夹中
  + 所有的volume sources必须位于同一个命名空间
  + 当前状态如下的volume sources可被打包
    + secret
    + downwardAPI
    + configMap
    + serviceAccountToken
+ quobyte
  + 由Quobyte提供的卷
+ rbd
  + 由ceph提供的Rados Block Device
+ secret
  + 用于传递敏感信息
+ storageOS
  + 由StorageOS提供的卷，StorageOS为商业化的存储解决方案商
+ vsphereVolume
  + vsphere卷
  
## Using subPath

subPath机制提供了同一个volume被pod中的不同container以不同子路径使用的机制

一个例子

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### Using subPath with expanded environment variables

可通过使用subPathExpr构建subPath名称，构建subPath名称时需要通过downward API environment variables。

subPath与subPathExpr是互斥的，只允许同时使用一种方式

一个例子

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

## Resources

资源管理

对于empty volume而言，存储媒体类型取决于存储kubelet root dir（通常为`/var/lib/kubelet`）的媒体类型。

对于emptyDir或者hostPath volume而言，关于存储空间的限制目前无限制，同时其在不同容器或不同pods间的隔离性并无。

## Out-of-tree volume plugins

out-of-tree volume包括两类：Container Storage Interface以及FlexVolume

### CSI

容器存储接口定义容器调度系统的标准化存储接口，通过该接口可将任意的存储系统导出至容器负载中，详细参考[CSI](../../../CNCF_spec/container_storage_interface.md)

一旦符合CSI的volume driver被部署，用户便可使用`csi volume type`对CSI driver导出的卷进行挂载。

对于csi volume而言，其使用方式有三种：

+ through a reference to a PersistentVolumeClaim
+ with a generic ephemeral volume (alpha feature)
+ with a CSI ephemeral volume is the driver support that (beta feature)

#### CSI raw block volume support

通过CSI驱动，可实现在Kubernetes 负载中使用raw block volume

ceph csi的部署实例参见[此处](../../kubeneters_cluster_manager/storage_manager/ceph_as_backend/Ceph_as_volume_backend.md)

#### CSI ephemeral volumes

可以在Pod specification中使用CSI volume，此时volumes为非持久化卷。

#### Migrating to CSI drivers from in-tree plugins

当`CSIMigration`特性被允许使用，现有的in-tree plugins的操作直接定向至CSI driver

### flexVolume

二进制形式的存储接口使用方式

## Mount propagation

mount 传播

Mount propagation是一种volume共享方式，允许volumes在同一个pod间或同一个node上的pod间进行共享。

卷的Mount propagation有mountPropagation域进行控制，该域存在于Container.volumeMounts中，可能存在的值有：

+ None 
  + This volume mount will not receive any subsequent mounts that are mounted to this volume or any of its subdirectories by the host. In similar fashion, no mounts created by the container will be visible on the host. This is the **default mode**.
  + This mode is equal to private mount propagation as described in the Linux kernel documentation
+ HostToContainer
  + This volume mount will receive all subsequent mounts that are mounted to this volume or any of its subdirectories. **In other words, if the host mounts anything inside the volume mount, the container will see it mounted there.**
+ Bidirectional
  + This volume mount behaves the same the HostToContainer mount. In addition, all volume mounts created by the container will be propagated back to the host and to all containers of all pods that use the same volume. **A typical use case for this mode is a Pod with a FlexVolume or CSI driver or a Pod that needs to mount something on the host using a hostPath volume.**
  + This mode is equal to rshared mount propagation as described in the Linux kernel documentation
