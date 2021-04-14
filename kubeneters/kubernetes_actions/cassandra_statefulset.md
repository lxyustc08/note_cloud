- [Deploying Cassandra with a StatefulSet](#deploying-cassandra-with-a-statefulset)
  - [Creating a headless Service for Cassandra](#creating-a-headless-service-for-cassandra)
  - [Using a StatefulSet to create a Cassandra ring](#using-a-statefulset-to-create-a-cassandra-ring)
    - [Validating the Cassandra StatefulSet](#validating-the-cassandra-statefulset)
    - [Modifying the Cassandra StatefulSet](#modifying-the-cassandra-statefulset)

# Deploying Cassandra with a StatefulSet

本部分将演示如何通过使用StatefulSet在Kubernetes上部署一个Apache Cassandra。

Cassandra一个数据库，需要使用持久化存储。本部分使用一个自定义的Cassandra种子提供器，让数据库集群在新的Cassandra加入时发现该实例。

## Creating a headless Service for Cassandra

创建如下的headless serivce，CassandraHeadlessService.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cassandra
  name: cassandra
  namespace: k8s-samples
spec:
  selector:
    app: cassandra
  ports:
  - port: 9042
  clusterIP: None
```

使用kubectl apply创建cassandra headless service

```
kubectl apply -f CassandraHeadlessService.yaml
```

验证该Headless Service创建状态

```
kubectl get svc -n k8s-samples
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
cassandra   ClusterIP   None         <none>        9042/TCP   8s
```

## Using a StatefulSet to create a Cassandra ring

使用下述配置文件创建一个Cassandra环CassandraStatefulSet.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
  labels:
    app: cassandra
  namespace: k8s-samples
spec:
  selector:
    matchLabels:
      app: cassandra
  serviceName: cassandra
  replicas: 3
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      terminationGracePeriodSeconds: 1800
      containers:
      - name: cassandra
        image: lxyustc.registrydomain.com:5000/cassandra:3.11
        imagePullPolicy:  Always
        ports:
        - containerPort: 7000
          name: intra-node
        - containerPort: 7001
          name: tls-intra-node
        - containerPort: 7199
          name: jmx
        - containerPort: 9042
          name: cql
        resources:
          limits:
             cpu: "500m"
             memory: 1Gi
          requests:
            cpu: "500m"
            memory: 1Gi
        securityContext:
          capabilities:
            add:
              - IPC_LOCK
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - nodetool drain
        env:
          - name: MAX_HEAP_SIZE
            value: 512M
          - name: HEAP_NEWSIZE
            value: 100M
          - name: CASSANDRA_SEEDS
            value: "cassandra-0.cassandra.k8s-samples.svc.cluster.local"
          - name: CASSANDRA_CLUSTER_NAME
            value: "K8Demo"
          - name: CASSANDRA_DC
            value: "DC1-K8Demo"
          - name: CASSANDRA_RACK
            value: "Rack1-K8Demo"
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        volumeMounts:
        - name: cassandra-data
          mountPath: /cassandra_data
  volumeClaimTemplates:
  - metadata:
      name: cassandra-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: csi-rbd-sc
```

该配置使用cassandra-0作为cassandra集群的种子。

### Validating the Cassandra StatefulSet

通过kubectl查看cassandra statefulset
   
```
kubectl get sts cassandra -n k8s-samples
NAME        READY   AGE
cassandra   3/3     16h
```

使用cassandra nodetool展示环状态

```
kubectl exec -n k8s-samples -it cassandra-0 -- nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.88.5.89   177.67 KiB  256          62.2%             ba71202b-3e0d-4f4c-ab99-a4f012acb2cc  rack1
UN  10.88.6.125  203.63 KiB  256          70.5%             c66ff4cf-5dc5-4e8d-8309-e1b74a97e6a5  rack1
UN  10.88.7.52   176.28 KiB  256          67.3%             33830dbd-3de6-4046-ad42-87642253ebcd  rack1
```

### Modifying the Cassandra StatefulSet

使用kubectl edit修改replicas为4

查看pods状态

```
kubectl get pods -w -n k8s-samples -l app=cassandra
NAME          READY   STATUS              RESTARTS   AGE
cassandra-0   1/1     Running             0          16h
cassandra-1   1/1     Running             0          16h
cassandra-2   1/1     Running             2          16h
cassandra-3   0/1     ContainerCreating   0          7s
cassandra-3   1/1     Running             0          14s
```

使用cassandra node工具查看cassandra集群状态

```
kubectl exec -n k8s-samples -it cassandra-0 -- nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.88.5.89   196.97 KiB  256          47.0%             ba71202b-3e0d-4f4c-ab99-a4f012acb2cc  rack1
UN  10.88.6.125  203.63 KiB  256          54.0%             c66ff4cf-5dc5-4e8d-8309-e1b74a97e6a5  rack1
UN  10.88.7.52   176.28 KiB  256          49.4%             33830dbd-3de6-4046-ad42-87642253ebcd  rack1
UN  10.88.7.53   75.85 KiB  256          49.6%             79a5a507-099f-41cd-a94a-5cc945fef1af  rack1
```
