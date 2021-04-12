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
