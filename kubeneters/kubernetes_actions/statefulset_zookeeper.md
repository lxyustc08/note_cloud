- [Running ZooKeeper, A Distributed System Coordinator](#running-zookeeper-a-distributed-system-coordinator)
  - [Creating a Zookeeper ensemble](#creating-a-zookeeper-ensemble)
    - [Facilitating leader election](#facilitating-leader-election)
    - [Achieving consensus](#achieving-consensus)
    - [Santiy testing the ensemble](#santiy-testing-the-ensemble)
    - [Providing durable storage](#providing-durable-storage)
  - [Ensuring consistent configuration](#ensuring-consistent-configuration)
    - [Configuring logging](#configuring-logging)
    - [Configuring a non-privileged user](#configuring-a-non-privileged-user)
  - [Managing the ZooKeeper process](#managing-the-zookeeper-process)
    - [Updating the ensemble](#updating-the-ensemble)
    - [Handing process failure](#handing-process-failure)
    - [Testing for liveness](#testing-for-liveness)
    - [Testing for readiness](#testing-for-readiness)
  - [Tolerating Node failure](#tolerating-node-failure)
    - [Surviving maintenance](#surviving-maintenance)

# Running ZooKeeper, A Distributed System Coordinator

本部分将通过使用StatefulSet机制部署一个Zookeeper集群，将了解以下知识点：

+ How to deploy a Zookeeper ensemble using StatefulSet
+ How to consistently configure the ensemble
+ How to spread the deployment of Zookeeper servers in the ensemble
+ How to use PodDisruptionBudgets to ensure service availability during planned maintenance

## Creating a Zookeeper ensemble

使用如下配置文件创建Zookeeper ensemble，zookeeper.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "lxyustc.registrydomain.com:5000/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
      storageClassName: csi-rbd-sc
```

使用kubectl部署zookeeper.yaml

```
kubectl apply -f zookeeper.yaml -n k8s-samples
```

查看zookeeper pod运行状态

```
kubectl get pods -n k8s-samples -l app=zk

NAME   READY   STATUS    RESTARTS   AGE
zk-0   1/1     Running   0          24m
zk-1   1/1     Running   0          23m
zk-2   1/1     Running   0          23m
```

确认zookeeper pod全部创建完成

### Facilitating leader election

对于zookeeper而言，只有在所有成员明确自己身份后才进行选举，可通过如下命令确认zookeeper的Pods的hostname

```
for i in 0 1 2; do kubectl exec zk-$i -n k8s-samples -- hostname; done

zk-0
zk-1
zk-2
```

该主机名是statefulSet controller自动设置。

zookeeper的每个节点通过myid设置其标识符，可通过如下命令获取每个节点的myid配置

```
for i in 0 1 2; do echo "myid zk-$i";kubectl exec zk-$i -n k8s-samples -- cat /var/lib/zookeeper/data/myid; done

myid zk-0
1
myid zk-1
2
myid zk-2
3
```

myid为zookeeper镜像中的自动化配置脚本script根据hostname自动创建生成。

可通过如下命令获取statefulSet中的Pod的Fully Qualified Domain Name（FQDN）

```
for i in 0 1 2; do kubectl exec zk-$i -n k8s-samples -- hostname -f; done

zk-0.zk-hs.k8s-samples.svc.cluster.local
zk-1.zk-hs.k8s-samples.svc.cluster.local
zk-2.zk-hs.k8s-samples.svc.cluster.local
```

其中`zk-hs.k8s-samples.svc.cluster.local`是由StatefulSet创建的域名，该域名对于StatefulSet中的所有Pod适用。

要获取zookeeper集群的配置信息，可使用如下命令查看：

```
kubectl exec zk-0 -n k8s-samples -- cat /opt/zookeeper/conf/zoo.cfg

#This file was autogenerated DO NOT EDIT
clientPort=2181
dataDir=/var/lib/zookeeper/data
dataLogDir=/var/lib/zookeeper/data/log
tickTime=2000
initLimit=10
syncLimit=5
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
autopurge.snapRetainCount=3
autopurge.purgeInteval=12
server.1=zk-0.zk-hs.k8s-samples.svc.cluster.local:2888:3888
server.2=zk-1.zk-hs.k8s-samples.svc.cluster.local:2888:3888
server.3=zk-2.zk-hs.k8s-samples.svc.cluster.local:2888:3888
```

上述配置文件也是zookeeper镜像中script脚本自动生成。

### Achieving consensus

共识协议的取得需要共识参与方的标识符完全唯一。在Zab共识协议中，共识协议参与方不应该声明同一个标识符。这是不同进程间对数据提交取得共识的前提。

通过Kubernetes StatefulSet机制，可以使StatefulSet中的zookeeper Pod获得不同的域名，通过这个不同的域名，可确保StatefulSet中的zookeeper Pod中的myid独立。

### Santiy testing the ensemble

对于Zookeeper而言最简单的测试方式使，将数据写入Zookeeper server并从另一个server中读取数据。

使用如下命令，将单词world写入zk-0的/hello路径下。

```
kubectl exec -n k8s-samples zk-0 -- zkCli.sh create /hello world

......
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
Created /hello
```

使用如下命令，从zk-1中获取数据

```
kubectl exec zk-1 -n k8s-samples -- zkCli.sh get /hello

......
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
cZxid = 0x100000002
world
ctime = Wed Apr 14 05:53:10 UTC 2021
mZxid = 0x100000002
mtime = Wed Apr 14 05:53:10 UTC 2021
pZxid = 0x100000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```

### Providing durable storage

zookeeper将所有的条目提交至持久化的WAL，并定期将内存中的快照写入存储媒体中。通过使用WAL提供的持久性，对于哪些利用共识协议实现复数状态机的应用而言，是一种常用技术。

本部分对由StatefulSet实现的zookeeper集群的持久化存储进行实验。

删除zk StatefulSet

```
kubectl delete statefulset zk -n k8s-samples
```

等待StatefulSet中的所有Pod删除后，重新部署zookeeper.yaml

```
kubectl apply -f zookeeper.yaml -n k8s-samples
```

等待StatefulSet中的所有Pod创建完成，从zk-2 Pod中重新查看/hello数据

```
kubectl exec -n k8s-samples zk-2 -- zkCli.sh get /hello

......
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
cZxid = 0x100000002
world
ctime = Wed Apr 14 05:53:10 UTC 2021
mZxid = 0x100000002
mtime = Wed Apr 14 05:53:10 UTC 2021
pZxid = 0x100000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```

本处利用了StatefulSet提供的pvc持久化绑定的机制，实现数据的持久化存储。

## Ensuring consistent configuration

zookeeper需要一致性配置来选举leader并组织形成法定人数。对于Zab协议而言，同样也需要一致性配置使协议在网络上可以正常工作。在示例部署文件中，直接将配置内嵌至template的command域中

```
kubectl get sts zk -n k8s-samples -o yaml

...
      containers:
      - command:
        - sh
        - -c
        - start-zookeeper --servers=3 --data_dir=/var/lib/zookeeper/data --data_log_dir=/var/lib/zookeeper/data/log
          --conf_dir=/opt/zookeeper/conf --client_port=2181 --election_port=3888 --server_port=2888
          --tick_time=2000 --init_limit=10 --sync_limit=5 --heap=512M --max_client_cnxns=60
          --snap_retain_count=3 --purge_interval=12 --max_session_timeout=40000 --min_session_timeout=4000
          --log_level=INFO
```

除了直接内嵌外，可以使用env进行配置。

### Configuring logging

对于zookeeper而言，可通过zkGenConfig.sh脚本控制Zookeeper的日志。Zookeeper使用Log4j，默认情况下，其使用基于时间以及日志大小的滚动文件追加程序进行日志记录配置。

使用如下的命令从StatefulSet的一个Pod中获取日志配置

```
kubectl exec zk-0 -n k8s-samples -- cat /usr/etc/zookeeper/log4j.properties
zookeeper.root.logger=CONSOLE
zookeeper.console.threshold=INFO
log4j.rootLogger=${zookeeper.root.logger}
log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.Threshold=${zookeeper.console.threshold}
log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L] - %m%n
```

上述配置是最为简单并安全地获取容器运行日志的方式，直接将容器运行日志输出至标准文件输出，这样可以通过Kubernetes log命令获取运行日志。

一个例子，使用kubectl获取pods的最后20条日志

```
kubectl logs -n k8s-samples zk-0 --tail 20
2021-04-14 06:52:00,723 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58252
2021-04-14 06:52:00,723 [myid:1] - INFO  [Thread-524:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58252 (no session established for client)
2021-04-14 06:52:08,370 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:58294
2021-04-14 06:52:08,370 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58294
2021-04-14 06:52:08,371 [myid:1] - INFO  [Thread-525:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58294 (no session established for client)
2021-04-14 06:52:10,713 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:58348
2021-04-14 06:52:10,714 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58348
2021-04-14 06:52:10,715 [myid:1] - INFO  [Thread-526:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58348 (no session established for client)
2021-04-14 06:52:18,386 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:58390
2021-04-14 06:52:18,387 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58390
2021-04-14 06:52:18,389 [myid:1] - INFO  [Thread-527:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58390 (no session established for client)
2021-04-14 06:52:20,739 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:58408
2021-04-14 06:52:20,739 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58408
2021-04-14 06:52:20,741 [myid:1] - INFO  [Thread-528:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58408 (no session established for client)
2021-04-14 06:52:28,381 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:58452
2021-04-14 06:52:28,382 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58452
2021-04-14 06:52:28,382 [myid:1] - INFO  [Thread-529:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58452 (no session established for client)
2021-04-14 06:52:30,702 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /127.0.0.1:58470
2021-04-14 06:52:30,702 [myid:1] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@883] - Processing ruok command from /127.0.0.1:58470
2021-04-14 06:52:30,703 [myid:1] - INFO  [Thread-530:NIOServerCnxn@1044] - Closed socket connection for client /127.0.0.1:58470 (no session established for client)
```

### Configuring a non-privileged user

可通过StatefulSet的template域中的SecurityContext域配置非特权用户

```yaml
securityContext:
  runAsUser: 1000
  fsGroup: 1000
```

通过kubectl命令验证非特权用户生效


```
kubectl exec zk-0 -n k8s-samples -- ps -elf
F S UID          PID    PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
4 S zookeep+       1       0  0  80   0 -  1126 wait   06:08 ?        00:00:00 sh -c start-zookeeper --servers=3 --data_dir=/var/lib/zookeeper/data --data_log_dir=/var/lib/zookeeper/data/log --conf_dir=/opt/zookeeper/conf --client_port=2181 --election_port=3888 --server_port=2888 --tick_time=2000 --init_limit=10 --sync_limit=5 --heap=512M --max_client_cnxns=60 --snap_retain_count=3 --purge_interval=12 --max_session_timeout=40000 --min_session_timeout=4000 --log_level=INFO
4 S zookeep+       7       1  0  80   0 - 962377 futex_ 06:08 ?       00:00:07 /usr/lib/jvm/java-8-openjdk-amd64/bin/java -Dzookeeper.log.dir=/var/log/zookeeper -Dzookeeper.root.logger=INFO,CONSOLE -cp /usr/bin/../build/classes:/usr/bin/../build/lib/*.jar:/usr/bin/../share/zookeeper/zookeeper-3.4.10.jar:/usr/bin/../share/zookeeper/slf4j-log4j12-1.6.1.jar:/usr/bin/../share/zookeeper/slf4j-api-1.6.1.jar:/usr/bin/../share/zookeeper/netty-3.10.5.Final.jar:/usr/bin/../share/zookeeper/log4j-1.2.16.jar:/usr/bin/../share/zookeeper/jline-0.9.94.jar:/usr/bin/../src/java/lib/*.jar:/usr/bin/../etc/zookeeper: -Xmx512M -Xms512M -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /usr/bin/../etc/zookeeper/zoo.cfg
4 R zookeep+    6511       0  0  80   0 -  8605 -      06:55 ?        00:00:00 ps -elf
```

StatefulSet中的Pod将对应的PVC挂载至路径/var/lib/zookeeper/data下，该路径通常情况只能被root访问，通过securityContext中的fsGroup配置选项，将该文件夹设置为组zookeeper所有，因此，可被StatefulSet中的Pod正常读写数据。

通过如下命令验证其权限属性

```
kubectl exec -ti zk-0 -n k8s-samples -- ls -ld /var/lib/zookeeper/data
drwxrwsr-x 4 zookeeper zookeeper 4096 Apr 14 02:52 /var/lib/zookeeper/data
```

## Managing the ZooKeeper process

zookeeper官方手册中说明，对于zookeeper集群而言，最好设置一个超级进程对zookeeper服务器集群进行管理，负责重启失效进程。

在Kubernetes中，无需部署超级进程，由Kubernetes本身充当超级进程的角色。

### Updating the ensemble

zookeeper集群使用StatefulSet机制部署，配置文件中已设置为`RollingUpdate`升级策略。使用kubectl patch命令对cpu资源进行配置升级。

```
kubectl patch sts zk --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value":"0.3"}]'
```

查看zookeeper集群升级状态

```
kubectl rollout status sts zk -n k8s-samples
waiting for statefulset rolling update to complete 0 pods at revision zk-6487dd676d...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 1 pods at revision zk-6487dd676d...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 2 pods at revision zk-6487dd676d...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
statefulset rolling update complete 3 pods at revision zk-6487dd676d...
```

使用kubectl rollout history命令查看升级历史以及历史配置

```
kubectl rollout history sts zk -n k8s-samples
statefulset.apps/zk
REVISION
1
2
```

使用kubectl rollout undo命令回滚升级修改

```
kubectl rollout undo sts zk -n k8s-samples
waiting for statefulset rolling update to complete 0 pods at revision zk-6d774745db...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 1 pods at revision zk-6d774745db...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 2 pods at revision zk-6d774745db...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
statefulset rolling update complete 3 pods at revision zk-6d774745db...
```

### Handing process failure

Restart Policies配置kubernetes控制进程失效的行为，该进程为Pod中容器的entry point进程。对于`StatefulSet`中的Pods而言，唯一合适的`RestartPolicy`是Always，该值也是RestartPolicy的默认值。对于stateful应用而言，该值永远不应该被覆盖。

使用如下命令查看zk-0 pods中运行的进程

```
kubectl exec -n k8s-samples  zk-0 -- ps -ef

UID          PID    PPID  C STIME TTY          TIME CMD
zookeep+       1       0  0 01:55 ?        00:00:00 sh -c start-zookeeper --servers=3 --data_dir=/var/lib/zookeeper/data --data_log_dir=/var/lib/zookeeper/data/log --conf_dir=/opt/zookeeper/conf --client_port=2181 --election_port=3888 --server_port=2888 --tick_time=2000 --init_limit=10 --sync_limit=5 --heap=512M --max_client_cnxns=60 --snap_retain_count=3 --purge_interval=12 --max_session_timeout=40000 --min_session_timeout=4000 --log_level=INFO
zookeep+       8       1  0 01:55 ?        00:00:02 /usr/lib/jvm/java-8-openjdk-amd64/bin/java -Dzookeeper.log.dir=/var/log/zookeeper -Dzookeeper.root.logger=INFO,CONSOLE -cp /usr/bin/../build/classes:/usr/bin/../build/lib/*.jar:/usr/bin/../share/zookeeper/zookeeper-3.4.10.jar:/usr/bin/../share/zookeeper/slf4j-log4j12-1.6.1.jar:/usr/bin/../share/zookeeper/slf4j-api-1.6.1.jar:/usr/bin/../share/zookeeper/netty-3.10.5.Final.jar:/usr/bin/../share/zookeeper/log4j-1.2.16.jar:/usr/bin/../share/zookeeper/jline-0.9.94.jar:/usr/bin/../src/java/lib/*.jar:/usr/bin/../etc/zookeeper: -Xmx512M -Xms512M -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /usr/bin/../etc/zookeeper/zoo.cfg
```

终止zk-0中的java进程

```
kubectl exec zk-0 -n k8s-samples -- pkill java
```

查看zk StatefulSet中pods状态

```
kubectl get pods -n k8s-samples -l app=zk -w
NAME   READY   STATUS    RESTARTS   AGE
zk-0   1/1     Running   0          10m
zk-1   1/1     Running   0          11m
zk-2   1/1     Running   0          12m
zk-0   0/1     Error     0          10m
zk-0   0/1     Running   1          10m
zk-0   1/1     Running   1          10m
```

If your application uses a script (such as zkServer.sh) to launch the process that implements the application's business logic, **the script must terminate with the child process**. This ensures that Kubernetes will restart the application's container when the process implementing the application's business logic fails。

### Testing for liveness

使用Kubernetes的liveness probes机制通知Kubernetes应用进程处于非健康状态，需要Kubernetes介入重启。

zk sts中配置了liveness probe如下：

```yaml
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "zookeeper-ready 2181"
```

该livenessProbe执行zookeeper-ready脚本

```shell
OK=$(echo ruok | nc 127.0.0.1 $1)
if [ "$OK" == "imok" ]; then
    exit 0
else
    exit 1
fi
```

使用命令观察zk StatefulSet中的Pods状态

```
kubectl get pod -w -l app=zk -n k8s-samples
```

执行命令删除zookeeper-ready脚本（/usr/bin/zookeeper-ready为/opt/zookeeper/bin/zookeeper-ready的软连接）

```
kubectl exec zk-0 -n k8s-samples -- rm /opt/zookeeper/bin/zookeeper-ready
```

此时zk-0状态如下

```
NAME   READY   STATUS    RESTARTS   AGE
zk-0   1/1     Running   1          21m
zk-1   1/1     Running   0          22m
zk-2   1/1     Running   0          23m
zk-0   0/1     Running   1          24m
zk-0   0/1     Running   2          24m
zk-0   1/1     Running   2          24m
```

由于删除了zookeeper-ready脚本，因此liveness probe无法获取到正常的2181端口范围的值，kubernetes认为该Pod失效，重启该pod。

### Testing for readiness

readiness（准备就绪）与liveness（存活）是不同的，readiness标识进程已经准备好接受输入处理请求，liveness标识进程处于存活状态，可以被调度且健康状态良好。liveness是readiness的必要不充分条件.

上述区别，在初始化与终止状态时十分重要，此时进程已经存在但不一定处于准备就绪阶段。

当用户指定了readniess probe时，kubernetes确保进程直到处于准备就绪时才接受网络请求。zk StatefulSet中配置了readniss

```yaml
readinessProbe:
  exec:
    command:
    - sh
    - -c
    - "zookeeper-ready 2181"
  initialDelaySeconds: 10
  timeoutSeconds: 5
```

尽管在zk StatefulSet中，liveness与readiness probe是一致的，但同时指定两者是十分重要的，通过这两个probe可确保仅有健康的Zookeeper ensembel才可接收网络流量。

## Tolerating Node failure

zookeeper需要通过服务器的仲裁保证数据更改成功提交。对于例子中部署的3节点ensemble而言，至少需要2个server处于健康运行状态。在基于仲裁的系统中，成员需要跨故障域进行部署以确保可用性。为避免单节点失效，最好的部署方式是避免一个应用中的多个协同实例部署在同一个机器上。

要实现上述目的，可以利用Kubernetes的节点亲和性进行配置。部署文件中节点亲和性配置如下：

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: "app"
              operator: In
              values:
              - zk
        topologyKey: "kubernetes.io/hostname"
```

The requiredDuringSchedulingIgnoredDuringExecution field tells the Kubernetes Scheduler that it should never co-locate two Pods which have app label as zk in the domain defined by the topologyKey. The topologyKey kubernetes.io/hostname indicates that the domain is an individual node. Using different rules, labels, and selectors, you can extend this technique to spread your ensemble across physical, network, and power failure domains

### Surviving maintenance

本部分主要针对节点失效进行实验。

使用kubectl获取PodDistruptionBudget

```
kubectl get pdb zk-pdb -n k8s-samples
NAME     MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
zk-pdb   N/A             1                 1                     24h
```

上述输出中，`max-unavailable`域说明zk StatefulSet中最多允许1个Pod处于unavailable状态。

获取各Pods当前运行节点

```
for i in 0 1 2; do kubectl get pod zk-$i -n k8s-samples --template {{.spec.nodeName}}; echo ""; done

worker-amd64-1
worker-amd64-gpuceph-node3
worker-amd64-gpuceph-node2
```

由于环境问题，此处不再做进一步实验，参照[此链接](https://v1-20.docs.kubernetes.io/docs/tutorials/stateful-application/zookeeper/#surviving-maintenance)即可。
