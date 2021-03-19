- [Persistent Volume](#persistent-volume)
  - [Lifecycle of a volume and claim](#lifecycle-of-a-volume-and-claim)
    - [Provisioning](#provisioning)
      - [Static](#static)
      - [Dynamic](#dynamic)
    - [Binding](#binding)
    - [Using](#using)
    - [Storage Object in Use Protection](#storage-object-in-use-protection)
    - [Reclaiming](#reclaiming)
      - [Retian](#retian)
      - [Delete](#delete)
      - [Recycle](#recycle)
    - [Reserving a PersistentVolume](#reserving-a-persistentvolume)
    - [Expanding Persistent Volumes Claims](#expanding-persistent-volumes-claims)
      - [CSI Volume expansion](#csi-volume-expansion)
      - [Resizing a volume containing a file system](#resizing-a-volume-containing-a-file-system)
      - [Resizing an in-use PersistentVolumeClaim](#resizing-an-in-use-persistentvolumeclaim)
      - [Recoving from Failure when Expanding Volumes](#recoving-from-failure-when-expanding-volumes)
  - [Persistent Volumes](#persistent-volumes)
    - [Capacity（规格）](#capacity规格)
    - [Volume Mode](#volume-mode)
    - [Access Modes](#access-modes)
    - [Class](#class)
    - [Reclaim Policy](#reclaim-policy)
    - [Mount Options](#mount-options)
    - [Node Affinity](#node-affinity)
    - [Phase](#phase)
  - [PersistentVolumeClaims](#persistentvolumeclaims)
    - [Access Modes](#access-modes-1)
    - [Volume Modes](#volume-modes)
    - [Resources](#resources)
    - [Selector](#selector)
    - [Class](#class-1)
  - [Claims As Volumes](#claims-as-volumes)
    - [A Note on Namespaces](#a-note-on-namespaces)
    - [PersistentVolumes typed hostPath](#persistentvolumes-typed-hostpath)
  - [Raw Block Volume Support](#raw-block-volume-support)
    - [Binding Block Volumes](#binding-block-volumes)
  - [Volume Snapshot and Restore Volume from Snapshot Support](#volume-snapshot-and-restore-volume-from-snapshot-support)
    - [Create a PersistentVolumeClaim from a Volume Snapshot](#create-a-persistentvolumeclaim-from-a-volume-snapshot)
  - [Volume Cloning](#volume-cloning)
    - [Create PersistentVolumeClaim from an existing PVC](#create-persistentvolumeclaim-from-an-existing-pvc)
  - [Writing Protable Configuration](#writing-protable-configuration)

# Persistent Volume

为了管理存储，Kubernetes引入两类API，Persistent Volume（PV）以及Persistent Volume Claim（PVC）

+ Persistent Volume，持久化卷
  + 持久化卷是集群中存储的一部分，有管理员配置或由`StorageClass`动态配置，PVs是volume的一个子类，其完全独立于Pod的生命周期；
  + PV的API负责捕获存储实现细节，诸如NFS，iSCSI，或者由云提供的存储
+ Persistent Volume Claim，持久化卷声明
  + 持久化卷生命是一个由用户对存储提出的请求
  + 持久化卷声明类似于Pods，Pods消耗节点资源（CPU、内存），PVC消耗PV资源
  + 持久化卷声明请求特定的存储大小以及访问模式（ReadWriteOnce、ReadOnlyMany或者ReadWriteMany）

实际应用过程中，需要屏蔽用户对于集群提供的PV的属性感知（性能，访问模式，PV的具体实现），集群管理员同时需要提供不同的PV；为满足上述需求，Kubernete提出StorageClass资源

## Lifecycle of a volume and claim

PVs是集群中的资源，**PVCs是对这些资源的请求，同时充当资源的声明检查**。这两者间的区别联系如下：

### Provisioning

对于PVs而言，其有两种调配方式，动态或者静态：

#### Static

由集群管理员创建一系列数量的PV，并交由用户进行使用。此种方式由管理员对PV的细节进行管理。

#### Dynamic

这种方式由集群为用户动态创建满足用户提交的PVC要求的PV。集群动态创建时基于StorageClass，PVC配置时需指定StorageClass，对于集群管理员而言，其需要预先创建StorageClass。若PVC定义是将StorageClass指定为`""`，则该PVC禁用动态存储调配。

开启Dynamic配置需允许Kubernetes API服务器允许`DefaultStorageClass`选项

### Binding

当PVC（Persistent Volume Claim）被创建后，master寻找一个匹配的PV（该PV已经存在的情况下）。对于动态配置情况而言，由集群动态创建的PV总是绑定到PVC上。

```
kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc   Bound    pvc-3625fe8a-cd60-40a6-a70a-17711dbeb512   1Gi        RWO            csi-rbd-sc     6m37
```

一旦PV绑定到PVC，该绑定关系是独立的。通过ClaimRef实现PVC与PV之间的双向绑定，绑定关系中PVC到PV的绑定是1对1映射。

```
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-3625fe8a-cd60-40a6-a70a-17711dbeb512   1Gi        RWO            Delete           Bound    default/rbd-pvc   csi-rbd-sc              6m33s
```

当无符合PVC的PV存在时，该绑定无限期处于非绑定状态直至满足PVC要求的PV被创建。比如，集群中配置了若干个50G的PVs，这些PVs无法满足要求100G的PVC，此时PVC出于unbound状态，直至集群创建一个100G的PV

### Using

Pods使用PVC作为数据卷，集群获取PVC的详细信息，获取绑定的PV，并将PV绑定至Pod上。此外对于支持多种访问模式的PVC而言，由用户通过PVC对PV的在Pod中的访问模式进行指定。

一旦用户拥有PVC，并且该PVC绑定了PV，则绑定的PV在用户所需的时长内永远属于用户。

用户通过在Pods的配置文件中指定`volumes`配置为`persistentVolumeClaim`对Pods进行调度。

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-rbd-demo-pod
spec:
  containers:
    - name: web-server
      image: docker.io/library/nginx:latest
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false
```

### Storage Object in Use Protection

为防止PV、PVC误删，Kubernetes提供了使用保护机制。该机制确保PVCs在被Pods使用，以及PV处于绑定状态下发生误删时，确保实际对象不被删除，防止数据丢失。

> PVC处于激活状态的定义：使用该PVC的Pod对象存在

上述保护机制有两点：

1. 如果一个用户删除一个被Pod使用的PVC，这个PVC不会被立马移除，该PVC移除操作被推迟至该PVC不被任何Pod使用
2. 如果一个用户删除一个被绑定的PV，该PV不会被立马移除，该PV的移除操作被推迟至该PV不被绑定至PVC

上述保护机制中，PVC由名称为`kubernetes.io/pvc-protection`的Finalizer进行保护，PV由名称为`kubernetes.io/pv-protection`的Finalizer进行保护。

### Reclaiming

当用户将他们的卷使用完成后，用户将其PVC删除，绑定至PVC的PV的回收策略告知集群，该PV在其绑定的PVC对象删除后如何进行处理。当前，由三种回收机制Retianed，Recycled以及Deleted。

动态创建的PV的回收策略继承自StorageClass，创建StorageClass的默认回收策略为`Delete`。在PV创建完成后集群管理员可对PV回收策略尽心进一步修改编辑。

#### Retian

使用`Retian`作为回收策略的PV通常用于手动的资源回收。当使用此策略的PV在其绑定的PVC被删除后，该PV仍然存在，状态被认为为"released"。此时的PV不能被其他的PVC绑定，因为此时的前一个PVC相关的数据仍然保留在该PV上。集群管理员可手动对PV进行如下处理：

1. 删除该PV。此时该PV关联的外部存储资源（AWS EBS、GCE PD、Azure Disk、Cinder volume）仍然保留；
2. 手动清除外部存储资源的收据；
3. 手动清除外部存储资源资源，或者对外部存储资源进行手动重复利用，创建一个新的PV；

#### Delete

使用`Delete`作为回收策略的PV在PVC删除后会有两个动作：

1. Kubernetes集群中的PV对象被删除；
2. PV关联的外部存储资源（AWS EBS、GCE PD、Azure Disk、Cinder volume）一并删除；

#### Recycle

新版本中该回收策略被废弃

### Reserving a PersistentVolume

通过预约机制可实现PVC绑定至特定的PV，而不通过控制平面进行。此种方式需要手动指定绑定双方对象名称，如下面例子所述：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # Empty string must be explicitly set otherwise default StorageClass will be set
  volumeName: foo-pv
  ...
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef: # claimRef point to foo-pvc
    name: foo-pvc
    namespace: foo
  ...
```

### Expanding Persistent Volumes Claims

对于PVCs的大小扩展目前已默认开启。下列类型的卷均支持卷扩展：

+ gcePersistentDisk
+ awsElasticBlockStore
+ Cinder
+ glusterfs
+ rbd
+ Azure File
+ Azure Disk
+ Portworx
+ FlexVolumes
+ CSI

对PVC而言，若要扩展PVC中的volume大小，仅需在PVC配置文件中指定更大的存储空间即可，指定后重新应用该配置文件，此时绑定至PVC的PV大小扩展，并不会创建新的PV。

```
Name:          rbd-pvc
Namespace:     default
StorageClass:  csi-rbd-sc
Status:        Bound
Volume:        pvc-3625fe8a-cd60-40a6-a70a-17711dbeb512
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: rbd.csi.ceph.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      2Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    csi-rbd-demo-pod
Events:
  Type     Reason                      Age   From                               Message
  ----     ------                      ----  ----                               -------
  Normal   Resizing                    2m9s  external-resizer rbd.csi.ceph.com  External resizer is resizing volume pvc-3625fe8a-cd60-40a6-a70a-17711dbeb512
  Warning  ExternalExpanding           2m9s  volume_expand                      Ignoring the PVC: didn't find a plugin capable of expanding the volume; waiting for an external controller to process this PVC.
  Normal   FileSystemResizeRequired    2m8s  external-resizer rbd.csi.ceph.com  Require file system resize of volume on node
  Normal   FileSystemResizeSuccessful  9s    kubelet                            MountVolume.NodeExpandVolume succeeded for volume "pvc-3625fe8a-cd60-40a6-a70a-17711dbeb512"
```

#### CSI Volume expansion

对于CSI volume而言，卷的扩展默认配置处于开启状态，处于开启状态下其同样需要特定的CSI驱动支持卷扩展。

#### Resizing a volume containing a file system

对于包含文件系统的卷而言，其可扩展的前提是该文件系统为XFS、Ext3或者Ext4。

对一个包含文件系统的卷中的文件系统进行扩展时，文件系统仅能在Pod使用以`ReadWrite`模式的PVC下进行。文件系统的扩展仅在Pod启动或Pod使用的底层文件系统支持在线扩展情况下进行。

对于FlexVolumes而言，其允许resize的前提是，驱动的`RequiresFSResize`属性被设置为`true`

#### Resizing an in-use PersistentVolumeClaim

这种情况下，用户无需删除并重创建一个Pod，或者部署一个使用现有PVC的deployment。任何被使用的PVC在使用其的Pod的文件系统扩展后扩展空间变为可用。

#### Recoving from Failure when Expanding Volumes

当Volumes在扩展过程中底层存储失效时，集群管理员可以手动恢复Persistent Volume Claim的状态，并退出resize请求。在集群管理员不进行手动处理情况下，resize请求将在controller的控制下重复尝试。

集群管理员应进行如下步骤操作：

1. Mark the PersistentVolume that is bound to the PersistentVolumeClaim with `Retain` reclaim policy
2. Delete the PVCs. Since PV has `Retain` reclaim policy - we will not lose any data when we recrete the PVC
3. Delete the `claimRef` entry from PV specs, so as new PVC can bind to it. This should make the PV `Available`.
4. Re-create the PVC with smaller size than PV and set `volumeName` field of the PVC to the name of the PV. This should bind new PVC to existing PV.
5. Don't forget to restore the reclaim policy of the PV

## Persistent Volumes

同Kubernetes Object一样，每个PV对象包含一个spec以及status，相应地被称为volume的specification以及status。同样的用户仅能对volume的specification进行设置。PersistentVolume对象名称必须符合DNS subdomain name格式要求。一个典型例子：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

> **Note:** Helper programs relating to the volume type may be required for consumption of a PersistentVolume within a cluster. In this example, the PersistentVolme is of type NFS and the helper program /sbin/mount.nfs is required to support the mounting of NFS filesystems.

### Capacity（规格）

通常而言，一个PV拥有特定的规格设置，该配置通过PV的`capacity`属性配置。

当前技术状态，规格设置中仅能设置资源的storage size。不久的将来将可设置IOPS、throughput等属性

### Volume Mode

该配置属性由Kubernetes v1.18版本引入，现已为stable状态

目前Kubernetes支持PersistentVolumes的两种模式：`Filesystem`以及`Block`，通过属性`volmueMode`指定。

`volumeMode`属性是一个可选的API参数。当`volumeMode`未指定时，`Filesystem`是其默认值。

当`volumeMode`指定为`Block`时，Pod将volume视为裸块设备raw block device，之上无任何文件系统。此模式下Pod具有更高的访问速度。此时Pod中的应用必须知道如何处理裸块设备。

### Access Modes

PV可以在Provider以其允许的方式挂载至宿主机上。通常情况下Provider为PV提供不同的access modes。如下所示：

+ ReadWriteOnce —— 该volume可以被单个节点挂载为读写模式
+ ReadOnlyMany —— 该volume可以被多个节点挂载为只读模式
+ ReadWriteMany —— 该volume可以被多个节点挂载为读写模式

在命令行工具中，access mode被缩写为

+ RWO - ReadWriteOnce
+ ROX - ReadOnlyMany
+ RWX - ReadWriteMany

> A volume can only be mounted using one access mode at a time, even if it supports many. For example, a GCEPersistentDisk can be mounted as ReadWirteOnce by a single node or ReadOnlyMany by many nodes, but not at the same time.

关于provider支持的access mode的表格，详参照此[链接](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)

### Class

一个PV对象拥有类型，该值通过`storageClassName`属性进行设置，值为某一StorageClass名称。一个拥有特定Class的PV仅可被请求同类型的PVC绑定。

一个无`storageClassName`属性的PV仅可被绑定到无类型的PVC绑定。

### Reclaim Policy

当前状态下recalim policies具有如下属性：

+ Retain -- manual recalmation
+ Recycle -- basic scrub (`rm -rf /thevolume/*`)
+ Delete -- associaed storage asset such as AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder volume is deleted

### Mount Options

当一个PV挂载到一个节点时，Kubernetes集群管理员可以指定额外的mount选项。每种volume类型得mount options存在区别，可查看各volume type得mount options。

Mount options通过`mountOptions`属性指定。以往使用`volume.beta.kubenretes.io/mount-options`指定mount options而非`mountOptions`，后续将被废除，不建议继续使用

### Node Affinity

节点亲和性，PV可通过设置节点亲和性来限制哪些node可被volume访问。使用了设置了节点亲和性的Pod将仅能被调度到符合PV节点亲和性的节点上去。

### Phase

一个volume具有如下阶段：

+ Available -- a free resource that is not yet bound to a claim
+ Bound -- the volume is bound to a claim
+ Released -- the claim has been deleted, but the resource is not yet reclaimed by the cluster
+ Failed -- the volume has failed its automatic reclamation

## PersistentVolumeClaims

对于PVC对象而言，其跟Kubernetes对象类似，由spec与status组成，用户同样只能对spec进行配置。PVC对象的名称必须符合DNS subdomain name的要求

一个典型PVC对象的yaml描述文件内容如下

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### Access Modes

此部分参照PV对象的Access Modes

### Volume Modes

此部分参照PV对象的Volume Modes

### Resources

与Pods类似，PVC同样可以请求特定的资源配额。对于PVC而言，此种请求为存储资源而非CPU或内存资源。

### Selector

PVC可指定label selector来对PV进行进一步过滤。只有拥有符合selector标签的volumes才可被绑定至claim。selector包含两部分：

+ `matchLabels` - the volume must have a label with this value
+ `matchExpressions` - a list of requirements made by specifying key, list of values, and operator that relates the key and values. Valid operators include In, NotIn, Exists, and DoesNotExist.

对于`matchLabels`以及`matchExpressions`而言，所有的requirements为and关系，必须按顺序匹配

### Class

只有跟PVC同`storageClass`的PV才可被与PVC绑定

当PVC指定`storageClass`为`""`时，只有类型同为`""`的PV才可与该PVC绑定。

> 注：当`storageClass`被忽略时其默认值在各个集群中可能不一致，取决于`DefaultStorageClass`选线是否开启。

+ If the admission plugin is turned on, the administrator may specify a default StorageClass. All PVCs that have no storageClassName can be bound only to PVs of that default. Specifying a default StorageClass is done by setting the annotation storageclass.kubernetes.io/is-default-class equal to true in a StorageClass object. If the administrator does not specify a default, the cluster responds to PVC creation as if the admission plugin were turned off. If more than one default is specified, the admission plugin forbids the creation of all PVCs.
+ If the admission plugin is turned off, there is no notion of a default StorageClass. All PVCs that have no storageClassName can be bound only to PVs that have no class. In this case, the PVCs that have no storageClassName are treated the same way as PVCs that have their storageClassName set to "".

当PVC除了指定Class外同时指定`selector`，selector与Class同样也是AND关系：也即类型为PVC的类型，且满足selector要求的PV才可被绑定至PVC中。

> 注：当前状态下若PVC指定了非空的`selector`，该PVC不支持拥有动态的配置的PV

过去使用`volume.beta.kubernetes.io/storage-class`来配置`storageClassName`属性，后续不建议使用，将被废除。

## Claims As Volumes

Pods通过claim来获取volume存储。该Claims必须与Pods存在于同一个namespace中。集群首先在Pods的namespace中寻找claim，找到后，集群寻找满足PVC要求的PV，然后将该PV挂载至Pod中。

一个例子

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### A Note on Namespaces

PVs的绑定是独家的，**PVs不是namespace objects，不属于任何namesapce**；由于PersistentVolumeClaims是namespace objects，因此若在同一个namespace中挂载同一个PV，该PV应该设置为"Many" modes(`ROX`, `RWX`)

### PersistentVolumes typed hostPath

一个类型为`hostPath`的PersistentVolume使用Node上的文件或者文件夹模拟network-attached storage。

## Raw Block Volume Support

支持Raw Block的provisioning参考此[链接](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)

一个使用Raw Block的Pod的例子如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

从上可以看出，当为Pod添加一个raw block device时，device path在container中指定而非Pod的mount path处指定。

### Binding Block Volumes

当用户通过域`volumeMode`为Block的PVC请求一个raw block volume时，PV与PVC的绑定关系参考此[链接](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#binding-block-volumes)

## Volume Snapshot and Restore Volume from Snapshot Support

Volume Snapshot仅支持out-of-tree CSI volume plugins

### Create a PersistentVolumeClaim from a Volume Snapshot

从volume snapshot创建PVC的例子

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Volume Cloning

Volume Cloning仅被CSI volume插件支持

### Create PersistentVolumeClaim from an existing PVC

从已存在的PVC中创建一个新的PVC例子

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Writing Protable Configuration

编写具备移植性的存储配置可参考如下规则

+ 在配置包中包含PersistentVolumeClaim（除Depolyments，ConfigMaps外）
+ 在配置中不要包含PersistentVolume，原因在于用户在实例化配置时可能无法拥有创建PV的权限
+ 给予用户对于storage class name的可选项
  + 如果用户提供存储类型名称，PVC将匹配集群已存在的存储类型名称
  + 如果用户未提供存储类型名称，PV将被设置为默认的存储类型名称，通常情况下集群将配置存储类型名称
+ 在使用过程中，观察PVC的状态，如果PVC状态一直为非绑定状态，这说明该集群不支持动态存储或者该集群不具备存储系统。
