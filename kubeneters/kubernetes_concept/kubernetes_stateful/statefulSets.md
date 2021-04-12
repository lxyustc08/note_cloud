- [StatefulSets in Kubernetes](#statefulsets-in-kubernetes)
  - [Using StatefulSets](#using-statefulsets)
  - [Limitations](#limitations)
  - [Components](#components)
    - [Pod Selector](#pod-selector)
    - [Pod Identity](#pod-identity)
      - [Ordinal Index](#ordinal-index)
      - [Stable Network ID](#stable-network-id)
      - [Stable Storage](#stable-storage)
      - [Pod Name Label](#pod-name-label)
  - [Deployment and Scaling Guarantees](#deployment-and-scaling-guarantees)
    - [Pod Management Policies](#pod-management-policies)
      - [OrderedReady Pod Management](#orderedready-pod-management)
      - [Paraller Pod Management](#paraller-pod-management)
  - [Update Strategies](#update-strategies)
    - [On Delete](#on-delete)
    - [Rolling Updates](#rolling-updates)
      - [Partitions](#partitions)
      - [Forced Rollback](#forced-rollback)

# StatefulSets in Kubernetes

StatefulSets管理pods的集合，并提供集合中pods的`顺序和唯一性`保证。

与deployment类似，StatefulSet基于完全相同的container spec。与deployment不同的在于，StatefulSet为每个pods维护一个sticky identity(粘性标识)。在StatefulSet中，每个pods由相同的spec创建而来，但pods之间不可互换，pods的标识在任何重新部署过程中保持持久存在。

> Unlike a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: eacho has a persistent identifier that it maintains across any rescheduling.

在负载需要持久化存储卷的场景下，StatefulSet是一种较为可行的解决方案。尽管StatefulSet中的单个pods容易失效，但持久化的Pod标识符让已存在的volumes与替换失效的pods的新pods的匹配更为容易。

## Using StatefulSets

StatefulSets的典型使用场景如下：

+ Stable, unique network identifiers
+ Stable, persistent storage
+ Ordered, graceful deployment and scaling
+ Ordered, automated rolling updates

在上述场景中的Pods调度重配置定义下，stable即persistence的同义词。若一个应用无需任何持久化标签或者顺序部署，删除，缩放；则无需使用StatefulSets。

## Limitations

对于StatefulSet而言，其有如下限制：

+ 对于给定的Pod而言，其storage要么是基于storage class提供的PersistentVolume配置器提供，要么是由管理员预先配置；
+ 删除或者缩容StatefulSet，不会删除StatefulSet关联的volumes，这种做法是用于保证数据安全性；
+ StatefulSet目前需要一个[Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)用于负责Pods的网络标识符，该Service需要用户负责创建；
+ StatefulSets在其删除时不提供任何的Pods终止保证。在删除一个StatefuleSet时，要获得顺序且优雅的Pods终止，最好的方式是先将StatefulSet缩容至0，然后再行删除；
+ 在使用默认的Pod管理策略来进行Rollig Updates时，StatefulSet有可能进入一个需要手动恢复的损坏状态；

## Components

一个典型的StatefulSet的配置如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

上述例子可分为如下部分：

+ 名称为nginx的Headless Service，用于处理网络域名；
+ 名称为web的StatefulSet，其spec中定义了3个nginx副本；
+ 名称为www的volumeClaimTemplates，用于提供由名称为my-storage-class提供的持久化存储卷

### Pod Selector

用户必须设置`.spec.selector`值与`.spec.template.metadata.labels`一致。

### Pod Identity

StatefulSet中的Pods的标识符由三部分组成：

+ ordinal
+ network identity
+ stable storage

StatefulSet中的Pod的标识符跟随Pods，不论该Pods被重新调度到哪个node上

#### Ordinal Index

对于StatefulSet中的N个Pods而言，每个Pod均被赋值一个顺序整数，从0到N-1。每个Pod的整数均不一致。

#### Stable Network ID

StatefulSet中的每个Pod通过StatefulSet的名称以及Pod的顺序号推导其hostname。其pod的hostname的构成模式为`$(statefulset name)-$(ordinal)`。上述例子将构建如下3个Pod名称

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          9m38s
web-1   1/1     Running   0          9m25s
web-2   1/1     Running   0          9m16s
```

StatefulSet通过使用Headless Service来处理Pods的域名。Headless Service的域名格式为`$(service name).$(namespace).svc.cluster.local`，其中`cluster.local`为集群域名。当Pod被创建后，其获得的子域名格式为`$(podname).$(governing service domain)`，此处的governing service domain及statefulset配置文件中的`serviceName`域。

> 注意：StatefulSet中的Pod主机名(hostname)与域名(domainname)不一致，具体的区别展示可查看下表：

|Cluster Domain|Service(ns/name)|StatefulSet(ns/name)|StatefulSet Domain|Pod DNS|Pod Hostname|
|:---:|:---:|:---:|:---:|:---:|:---:|
|cluster.local|default/nginx|default/web|nginx.default.svc.cluster.local|web-{0..N-1}.nginx.default.svc.cluster.local|web-{0..N-1}|
|cluster.local|foo/nginx|foo/web|nginx.foo.svc.cluster.local|web-{0..N-1}.nginx.foo.svc.cluster.local|web-{0..N-1}|
|kube.local|foo/nginx|foo/web	|nginx.foo.svc.kube.local|web-{0..N-1}.nginx.foo.svc.kube.local|web-{0..N-1}|

#### Stable Storage

对于每个VolumeClaimTemplate，Kuberentes创建一个PersistentVolume。在上述例子中，每个Pod获得一个从名称为my-storage-class中创建的大小为1GB的pv。若配置文件未指定storage class，则使用集群默认的storage class。

当一个Pod被重新调度到另一个节点后，Pod的`volumeMounts`重新通过其PersistentVolume Claims挂载原PersistentVolumes。

> 注：StatefulSet中的Pod删除或StatefulSet被删除后，其Pod相关的PersistentVolume Claims不会被自动删除，需要手动释放。

#### Pod Name Label

当通过StatefulSet创建一个Pod时，该Pod被添加一个label，label形式为`statefulset.kubernetes.io/pod-name`。上述例子中的Pod如下所示

```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
web-0   1/1     Running   0          28m   app=nginx,controller-revision-hash=web-fff8657cc,statefulset.kubernetes.io/pod-name=web-0
web-1   1/1     Running   0          28m   app=nginx,controller-revision-hash=web-fff8657cc,statefulset.kubernetes.io/pod-name=web-1
web-2   1/1     Running   0          28m   app=nginx,controller-revision-hash=web-fff8657cc,statefulset.kubernetes.io/pod-name=web-2
```

## Deployment and Scaling Guarantees

对于StatefulSet而言，其部属以及扩缩容遵循如下规则：

+ 顺序创建，For a StatefulSet with N replicas, when Pods are being deployed, they are created sequentially, in order from {0..N-1}.
+ 逆序删除，When Pods are being deleted, they are terminated in reverse order, from {N-1..0}.
+ 顺序扩容，Before a scaling operation is applied to a Pod, all of its predecessors must be Running and Ready.
+ 逆序缩容，Before a Pod is terminated, all of its successors must be completely shutdown.

对于StatefulSet中的Pod而言，不建议其`pod.Spec.TerminationGracePeriodSeconds`配置为0

### Pod Management Policies

Kubernetes 1.7后，StatefulSet允许通过域`.spec.podManagementPolicy`放宽顺序性保证，并保留标识符独立性。

#### OrderedReady Pod Management

`OrderedReady`是StatefulSets的默认管理配置，其行为如上所述。

#### Paraller Pod Management

`Parallel` Pod management通知StatefulSet controller以并行方式同时启动或终止所有的pods，无需考虑pods之间启动终止顺序。该选项仅影响伸缩行为，**升级行为并不影响**。

## Update Strategies

在Kubernetes 1.7之后，StatefulSet提供域`.spec.updateStrategy`配置，该配置允许用户编辑或禁止Pod的containers, labels, resource request/limits, annotations自动升级规则。

### On Delete

`OnDelete`配置为传统的升级策略，此策略下，StatefulSet并不会自动升级集合中的Pods，用户必须手动删除Pods以触发StatefulSet controller以新配置创建Pod，从而实现Pod的升级目的。

### Rolling Updates

`RollingUpdate`为默认缺省配置，当StatefulSet的`.spec.updateStrategy.type`被配置为`RollingUpdate`时，StatefulSet controller将删除并重新创建StatefulSet中的Pod。对于StatefulSet中的Pod处理顺序与StatefulSet的Pod终止顺序一致（逆序，从大到小），每次升级一个Pod。

#### Partitions

当升级策略配置为`RollingUpdate`时，升级策略还可进一步配置为partitioned，通过设置`.spec.updateStrategy.rollingUpdate.partition`可将升级策略设置为部分升级。此时大于或等于某一顺序号的pod将自动升级，小于该顺序号的pod维持不变，即使手动删除。

#### Forced Rollback

When using Rolling Updates with the default Pod Management Policy (OrderedReady), it's possible to get into a broken state that requires manual intervention to repair.

If you update the Pod template to a configuration that never becomes Running and Ready (for example, due to a bad binary or application-level configuration error), StatefulSet will stop the rollout and wait.

In this state, it's not enough to revert the Pod template to a good configuration. Due to a known issue, StatefulSet will continue to wait for the broken Pod to become Ready (which never happens) before it will attempt to revert it back to the working configuration.

After reverting the template, you must also delete any Pods that StatefulSet had already attempted to run with the bad configuration. StatefulSet will then begin to recreate the Pods using the reverted template
