# StatefulSet Basic

本部分对StatefulSet的基本操作进行试实验

## Creating a StatefulSet

配置文件如[链接](kubernetes_actions_workspace/lxyustc-statefulset.yaml)

## Pods in a StatefulSet

StatefulSet中的每个Pod拥有独立的顺序标签以及持久化网络标识符

### Examining the Pod's Ordinal Index

输入如下命令获取StatefulSet's pods:

```
kubectl get pods -l app=nginx -n k8s-samples
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          16h
web-1   1/1     Running   0          16h
web-2   1/1     Running   0          16h
```

### Using Stable Network Identities

使用如下命令获取Pod的主机名称

```
for i in `seq 0 2`; do kubectl exec web-$i -n k8s-samples -- sh -c 'hostname'; done
```

```
web-0
web-1
web-2
```

通过busybox的`nslookup`命令获取Pods的dnsname。

运行如下命令，创建一个busybox shell

```
kubectl run -i --tty --image lxyustc.registrydomain.com:5000/busybox:1.28 dns-test --restart=Never --rm -n k8s-samples
```

在busybox shell中运行nslookup命令

```
nslookup web-0.nginx
```

输出如下

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.88.6.101 web-0.nginx.k8s-samples.svc.cluster.local
```

退出shell，删除StatefulSet的所有Pods

```
kubectl delete pod -l app=nginx -n k8s-samples
```

等待所有的Pods被重新创建，重新获取Pods主机名称

```
for i in `seq 0 2`; do kubectl exec web-$i -n k8s-samples -- sh -c 'hostname'; done
```

```
web-0
web-1
web-2
```

重新启动一个busybox shell

```
kubectl run -i --tty --image lxyustc.registrydomain.com:5000/busybox:1.28 dns-test --restart=Never --rm -n k8s-samples
```

再次查看Pods dns

```
nslookup web-0.nginx
```

输出如下：

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.88.6.104 web-0.nginx.k8s-samples.svc.cluster.local
```

通过上述实验可以发现，Pods的顺序号，主机名，SRV records，以及A record name并未发生改变，但Pods的IP地址发生了改变，因此在使用过程中应该时应该避免其他应用通过IP与StatefulSet中的Pods连接。

如果用户需要寻找并连接StatefulSet的活动成员，用户可以查询headless service的CNAME，该headless service的CNAME仅会包含当前StatefulSet中处于running和Ready状态的Pods。

如果用户应用已经实现了连接存活性逻辑，用户也可直接使用SRV records（web-0.nginx.k8s-samples.svc.cluster.local，web-1.nginx.k8s-samples.svc.cluster.local，web-2.nginx.k8s-samples.svc.cluster.local）。

### Writing to Stable Storage

查看StatefulSet的持久化存储状态

```
kubectl get pvc -l app=nginx -n k8s-samples
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-da6501da-d163-40f3-8e0c-f8f557cea514   1Gi        RWO            csi-rbd-sc     17h
www-web-1   Bound    pvc-81c91f5f-a503-4580-817a-6e66080d696a   1Gi        RWO            csi-rbd-sc     17h
www-web-2   Bound    pvc-bb6c1ce5-a940-4c18-b854-683fb04adf1f   1Gi        RWO            csi-rbd-sc     17h
```

可观察到，对于StatefulSet中的每个Pod均创建了对应的pvc，这些pvc根据StatefulSet的配置文件，挂载至Pod中的/usr/share/nginx/html文件夹上。

将Pod的主机名写入Pod的index.html中

```
for i in `seq 0 2`; do kubectl exec web-$i -n k8s-samples -- sh -c 'echo "$(hostname)" > /usr/share/nginx/html/index.html'; done
```

检测主机名是否写入成功

```
for i in `seq 0 2`; do kubectl exec -i -t web-$i -n k8s-samples -- curl http://localhost/; done
web-0
web-1
web-2
```

此时已将各Pod主机名写入各Pod的/usr/share/nginx/html/index.html中。

删除StatefulSet的所有Pods

```
kubectl delete pod -l app=nginx
```

等待所有的Pods重新创建完成，重新检查各Pod的/usr/share/nginx/html/index.html

```
for i in `seq 0 2`; do kubectl exec -i -t web-$i -n k8s-samples -- curl http://localhost/; done
web-0
web-1
web-2
```

可以发现即使Pods被删除重新创建后，原pvc仍可正确绑定至新创建的Pods中。

## Scalling a StatefulSet

与Deployment类似，通过修改StatefulSet的`replicas`域来对StatefulSet进行伸缩。可通过`kubectl scale`或`kubectl patch`进行，亦或者修改StatefulSet的配置文件，通过kubectl apply进行缩扩容。

### Scalling up

将replicas设置为5

```
kubectl scale sts web --replicas=5 -n k8s-samples
```

查看扩容后结果

```
kubectl get pods -n k8s-samples -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          14m
web-1   1/1     Running   0          14m
web-2   1/1     Running   0          14m
web-3   1/1     Running   0          2m16s
web-4   1/1     Running   0          2m1s
```

### Scalling down

将replicas设置为3

```
kubectl patch sts web -p '{"spec":{"replicas":3}}' -n k8s-samples
```

查看缩容过程

```
kubectl get pods -w -l app=nginx -n k8s-samples
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          16m
web-1   1/1     Running   0          16m
web-2   1/1     Running   0          16m
web-3   1/1     Running   0          4m35s
web-4   1/1     Running   0          4m20s
web-4   1/1     Terminating   0          4m33s
web-4   0/1     Terminating   0          4m36s
web-4   0/1     Terminating   0          4m45s
web-4   0/1     Terminating   0          4m45s
web-3   1/1     Terminating   0          5m1s
web-3   0/1     Terminating   0          5m3s
web-3   0/1     Terminating   0          5m9s
web-3   0/1     Terminating   0          5m9s
```

从缩容过程来看，缩容为逆序缩容

#### Persistent storage

查看此时的pvc

```
kubectl get pvc -l app=nginx -n k8s-samples
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-da6501da-d163-40f3-8e0c-f8f557cea514   1Gi        RWO            csi-rbd-sc     20h
www-web-1   Bound    pvc-81c91f5f-a503-4580-817a-6e66080d696a   1Gi        RWO            csi-rbd-sc     20h
www-web-2   Bound    pvc-bb6c1ce5-a940-4c18-b854-683fb04adf1f   1Gi        RWO            csi-rbd-sc     20h
www-web-3   Bound    pvc-79c1e84d-cdc9-4fc5-bce2-8fd7a48af9a7   1Gi        RWO            csi-rbd-sc     10m
www-web-4   Bound    pvc-284739a4-81fc-4660-b41d-f225b210fcb2   1Gi        RWO            csi-rbd-sc     9m52s
```

观察到，即使是StatefulSet缩容导致的Pod被删除，其原始挂载的pvc仍然存在并未被删除。

## Updating StatefulSets

从Kubernetes 1.7版本后，StatefulSet controller支持自动升级，并提供`spec.updateStrategy`配置项。

### Rolling Update

该选项对StatefulSet中的Pod以逆序的方式升级。

将StatefulSet的升级策略更改为RollingUpdate

```
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}' -n k8s-samples
```

修改容器镜像

```
kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"lxyustc.registrydomain.com:5000/nginx-slim:0.9"}]' -n k8s-samples
```

观察此时StatefulSet中Pods状态

```
kubectl get pods -w -l app=nginx -n k8s-samples
```

```
NAME    READY   STATUS              RESTARTS   AGE
web-0   1/1     Running             0          55m
web-1   1/1     Running             0          54m
web-2   0/1     ContainerCreating   0          3s
web-2   1/1     Running             0          16s
web-1   1/1     Terminating         0          55m
web-1   0/1     Terminating         0          55m
web-1   0/1     Terminating         0          55m
web-1   0/1     Terminating         0          55m
web-1   0/1     Pending             0          0s
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   1/1     Running             0          10s
web-0   1/1     Terminating         0          55m
web-0   0/1     Terminating         0          55m
web-0   0/1     Terminating         0          55m
web-0   0/1     Terminating         0          55m
web-0   0/1     Pending             0          0s
web-0   0/1     Pending             0          0s
web-0   0/1     ContainerCreating   0          0s
web-0   1/1     Running             0          10s
```

观察到RollingUpdate策略下，StatefulSet中的Pods逆序升级。

升级完成后查看container images：

```
for p in 0 1 2; do kubectl get pod "web-$p" -n k8s-samples --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
```

```
lxyustc.registrydomain.com:5000/nginx-slim:0.9
lxyustc.registrydomain.com:5000/nginx-slim:0.9
lxyustc.registrydomain.com:5000/nginx-slim:0.9
```

观察到镜像升级成功。

#### Staging an Update

本部分是对partition RollingUpdate的实验

更改web的升级策略

```
kubectl patch statefulset web -n k8s-samples -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
```

修改contianer images

```
kubectl patch statefulset web --type='json' -n k8s-samples -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"lxyustc.registrydomain.com:5000/nginx-slim:0.8"}]'
```

由于parition设置为3，因此web-0，web-1，web-2均不会升级，进行实验，手动删除web-2

```
kubectl delete pod web-2 -n k8s-samples
```

等待web-2重新创建完成后，查看web-2的container image

```
kubectl get pod web-2 -n k8s-samples --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'

lxyustc.registrydomain.com:5000/nginx-slim:0.9
```

发现并未改变

#### Rolling Out a Canary

本部分通过修改上一部分的partition参数进行进一步实验，将partition修改为2

```
kubectl patch statefulset web -n k8s-samples -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
```

查看StatefulSet中pods状态：

```
kubectl get pods -w -l app=nginx -n k8s-samples
NAME    READY   STATUS        RESTARTS   AGE
web-0   1/1     Running       0          40m
web-1   1/1     Running       0          41m
web-2   0/1     Terminating   0          14m
web-2   0/1     Terminating   0          14m
web-2   0/1     Terminating   0          14m
web-2   0/1     Pending       0          0s
web-2   0/1     Pending       0          0s
web-2   0/1     ContainerCreating   0          0s
web-2   1/1     Running             0          6s
```

可以发现web-2进行了升级，此时查看web-2的container image

```
kubectl get pod web-2 -n k8s-samples --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'

lxyustc.registrydomain.com:5000/nginx-slim:0.8
```

发现被成功修改为0.8版本

删除web-1 pod

```
kubectl delete pod web-1 -n k8s-samples
```

等待web-1重新创建后，查看web-1的镜像

```
kubectl get pod web-1 -n k8s-samples --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'

lxyustc.registrydomain.com:5000/nginx-slim:0.9
```

观察到web-1镜像并未更改，仍未0.9版本

#### Phased Roll Outs

通过上述修改replicas，某种程度上实现了金丝雀升级。同样的，分阶段修改replicas，亦可实现phased roll outs，比如将replicas修改为0，此时，StatefulSet中的所有Pod将进行升级

```
kubectl patch statefulset web -n k8s-samples -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
```

```
kubectl get pods -w -l app=nginx -n k8s-samples

NAME    READY   STATUS              RESTARTS   AGE
web-0   1/1     Running             0          57m
web-1   0/1     ContainerCreating   0          5s
web-2   1/1     Running             0          16m
web-1   1/1     Running             0          18s
web-0   1/1     Terminating         0          57m
web-0   0/1     Terminating         0          57m
web-0   0/1     Terminating         0          58m
web-0   0/1     Terminating         0          58m
web-0   0/1     Pending             0          0s
web-0   0/1     Pending             0          0s
web-0   0/1     ContainerCreating   0          0s
web-0   1/1     Running             0          18s
```

获取Pods中的container images

```
for p in 0 1 2; do kubectl get pod "web-$p" -n k8s-samples --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done

lxyustc.registrydomain.com:5000/nginx-slim:0.8
lxyustc.registrydomain.com:5000/nginx-slim:0.8
lxyustc.registrydomain.com:5000/nginx-slim:0.8
```

## Deleting StatefulSets

对于StatefulSets的删除而言，其支持两种方式Non-Cascading以及Cascading。前者仅StatefulSets被删除，但StatefulSets中的Pods并未删除；后者StatefulSets以及StatefulSet中的Pods全部删除。

### Non-Cascading Delete

删除web StatefulSet，使用Non-Cascading删除方式

```
kubectl delete statefulset web --cascade=false -n k8s-samples
```

查看此时的Pods

```
kubectl get pods -l app=nginx -n k8s-samples
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          15m
web-1   1/1     Running   0          16m
web-2   1/1     Running   0          32m
```

可发现原StatefulSet中的Pods仍然是存在的。

删除Pods

```
kubectl delete pods -n k8s-samples web-0
```

```
kubectl get pods -l app=nginx -n k8s-samples

NAME    READY   STATUS    RESTARTS   AGE
web-1   1/1     Running   0          22m
web-2   1/1     Running   0          38m
```

由于StatefulSet被删除，此时的web-0被删除后，并不会自动重新创建。

重新部署StatefulSet的配置文件

重新测试各Pod中的index.html文件

```
for i in `seq 0 2`; do kubectl exec -n k8s-samples -i -t "web-$i" -- curl http://localhost/; done

web-0
web-1
web-2
```

可发现pvc仍然成功挂载至web-0上。

### Cascading Delete

使用cascading删除方式删除web StatefulSet。

```
kubectl delete sts web -n k8s-samples
```

删除headless service nginx

```
kubectl delete svc -n k8s-samples nginx
```

重新部署web StatefulSet

等待所有Pods创建完成后，再次实验index.html是否变更。

```
for i in `seq 0 2`; do kubectl exec -n k8s-samples -i -t "web-$i" -- curl http://localhost/; done

web-0
web-1
web-2
```

可观察到，原有的pvc仍然被正确地挂载到新创建的Pod中。

## Pod Management Policy

为方便某些分布式系统仅需独立标识符而无需顺序性保障需求，从Kubernetes 1.7版本后，Kubernetes引入`.spec.podManagementPolicy`配置

### OrderedReady Pod Management

`OrderedReady`是`.spec.podManagementPolicy`的缺省配置。

### Parallel Pod Management

`Parallel` Pod管理策略告知StatefulSet controller以并行方式加载并终止所有的Pods，无需等待前后Pod启动或终止。**该策略仅影响伸缩行为，不影响升级动作**。

部署以下的StatefulSet

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
  serviceName: "nginx"
  podManagementPolicy: "Parallel"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: lxyustc.registrydomain.com:5000/nginx-slim:0.8
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
      resources:
        requests:
          storage: 1Gi
```

查看Pods状态

```
kubectl get pods -w -l app=nginx -n k8s-samples

NAME    READY   STATUS    RESTARTS   AGE
web-0   0/1     Pending   0          0s
web-0   0/1     Pending   0          0s
web-1   0/1     Pending   0          0s
web-1   0/1     Pending   0          0s
web-0   0/1     ContainerCreating   0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   1/1     Running             0          10s
web-0   1/1     Running             0          10s
```

可发现pod创建行为存在明显不同。两个pod web-0 web-1同时启动。

将StatefulSet的replicas设置为4

```
kubectl scale sts web -n k8s-samples --replicas=4
```

查看此时StatefulSet的Pod状态

```
kubectl get pods -w -l app=nginx -n k8s-samples

NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          7m50s
web-1   1/1     Running   0          7m50s
web-2   0/1     Pending   0          0s
web-3   0/1     Pending   0          0s
web-2   0/1     Pending   0          0s
web-3   0/1     Pending   0          0s
web-3   0/1     ContainerCreating   0          0s
web-2   0/1     ContainerCreating   0          0s
web-2   1/1     Running             0          17s
web-3   1/1     Running             0          18s
```

可观察到，StatefulSet扩容的两个pod，web-2 web-3同时创建。

