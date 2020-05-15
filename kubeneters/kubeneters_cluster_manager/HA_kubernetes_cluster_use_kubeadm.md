- [Use Kubernetes to build HA kubernetes cluster](#use-kubernetes-to-build-ha-kubernetes-cluster)
  - [Stacked control plane and etcd nodes](#stacked-control-plane-and-etcd-nodes)
    - [Basic environment](#basic-environment)
      - [kubernetes cluster 节点](#kubernetes-cluster-%e8%8a%82%e7%82%b9)
      - [Loadbalance 节点](#loadbalance-%e8%8a%82%e7%82%b9)
      - [Loadbalance 配置](#loadbalance-%e9%85%8d%e7%bd%ae)
    - [Create HA cluster](#create-ha-cluster)
      - [Install Container Runtime](#install-container-runtime)
      - [Add The first control plane](#add-the-first-control-plane)
      - [Add The second control plane](#add-the-second-control-plane)
      - [Add the third control plane](#add-the-third-control-plane)
      - [查看负载均衡器运行状态](#%e6%9f%a5%e7%9c%8b%e8%b4%9f%e8%bd%bd%e5%9d%87%e8%a1%a1%e5%99%a8%e8%bf%90%e8%a1%8c%e7%8a%b6%e6%80%81)
      - [查看集群节点信息](#%e6%9f%a5%e7%9c%8b%e9%9b%86%e7%be%a4%e8%8a%82%e7%82%b9%e4%bf%a1%e6%81%af)
    - [Test HA cluster](#test-ha-cluster)
      - [关闭master-arm-1](#%e5%85%b3%e9%97%admaster-arm-1)
      - [恢复master-arm-1](#%e6%81%a2%e5%a4%8dmaster-arm-1)

# Use Kubernetes to build HA kubernetes cluster

## Stacked control plane and etcd nodes

此种模式下etcd与控制平面部署在一起不使用单独的etcd集群。

### Basic environment

#### kubernetes cluster 节点

|节点名称|操作系统|硬件架构|容器运行环境|IP地址|
|:---:|:---:|:---:|:---:|:---:|
|master-arm-1|ubuntu19.10|raspberry pi4 </br> arm64|crio 1.18.0|10.10.197.98|
|master-arm-2|ubuntu19.10|raspberry pi4 </br> arm64|crio 1.18.0|10.10.197.97|
|master-arm-3|ubuntu19.10|raspberry pi4 </br> arm64|crio 1.18.0|10.10.197.99|

#### Loadbalance 节点

|节点名称|操作系统|硬件架构|IP地址|VIP|
|:---:|:---:|:---:|:---:|:---:|
|master-x86-1|ubuntu 20.04|amd64|10.10.197.96|10.10.197.101|

使用LVS与keepalived作为负载均衡器与健康检查器

#### Loadbalance 配置

采用LVS DR直接路由模式，关于直接路由模式的介绍，参照此[链接](http://www.linuxvirtualserver.org/VS-DRouting.html)。

keepalived.conf配置如下

```conf
! Configuration File for keepalived
global_defs {
    router_id loadbalance-director
}

vrrp_instance loadbalance {
    state MASTER
    interface eno1
    virtual_router_id 254
    priority 100
    advert_int 1

    virtual_ipaddress {
        10.10.197.101
    }
}

virtual_server 10.10.197.101 6443 {
    delay_loop 6
    lvs_sched wrr
    lvs_method DR
    protocol TCP
    real_server 10.10.197.98 6443 {
        weight 10
        TCP_CHECK {
           connect_timeout 10
           retry 3
           delay_before_retry 3
           connect_port 6443
        }
    }
   real_server 10.10.197.99 6443 {
        weight 10
        TCP_CHECK {
           connect_timeout 10
           retry 3
           delay_before_retry 3
           connect_port 6443
        }
   }
   real_server 10.10.197.97 6443 {
        weight 10
        TCP_CHECK {
           connect_timeout 10
           retry 3
           delay_before_retry 3
           connect_port 6443
        }
   }
}
```

使用LVS的DR模式需要对负载均衡集群中的真实机器进行配置，将负载均衡器VIP设置为回环网络接口IP alias，并对真实机器的ARP包响应行为进行限制。具体方法如下：

1. 编写VIP_up.sh 与 VIP_down.sh两个脚本：  
   
   VIP_up.sh

   ```shell
   # This shell is for set up loadbalance vip from client side

   #!/bin/bash

   SNS_VIP=10.10.197.101
   ifconfig lo:0 $SNS_VIP netmask 255.255.255.255 broadcast $SNS_VIP
   route add -host $SNS_VIP dev lo:0
   echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
   echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
   echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
   echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
   echo "SNS_VIP set up success"
   ```

   VIP_down.sh

   ```shell
   # This shell is used for set down loadbalance VIP

   SNS_VIP=10.10.197.101

   ifconfig lo:0 down
   route del $SNS_VIP
   echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
   echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
   echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
   echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
   echo "VIP set down sucess"
   ```

2. 将俩脚本放置在文件夹/usr/local/share/loadbalance_dr/下

3. 在文件夹/usr/lib/systemd/system下编写服务文件realServer.service
   
   ```service
   [Unit]
   Description=Set UP realServer
   Requires=network.target
   After=network.target

   [Service]
   Type=oneshot
   RemainAfterExit=yes
   ExecStart=bash /usr/local/share/loadbalance_dr/VIP_up.sh
   ExecStop=bash /usr/local/share/loadbalance_dr/VIP_down.sh

   [Install]
   WantedBy=multi-user.target
   ```

4. 通过realServer.service服务文件实现开机自动配置回环网络接口IP别名为VIP，并设置相关路由表条目

### Create HA cluster

创建HA时先仅在一个真实机器上对VIP进行配置，也即运行`systemctl start realServer`，其他节点暂不配置。

#### Install Container Runtime

此部分内容参考[CRI-O as container runtime](CRI-O_as_container_runtime.md)，此部分不在赘述

#### Add The first control plane

使用命令添加第一个control plane

```terminal
kubeadm init --image-repository=lxyustc.registrydomain.com:5000/google_containers --pod-network-cidr=10.88.0.0/16 --control-plane-endpoint="10.10.197.101:6443" --upload-certs
```

**注意:** CRI-O使用的cgroup manager为`systemd`而非`cgroupfs`，因此在上述命令执行过程中需配置`kubelet`配置文件/var/lib/kubelet/config.yaml，添加如下配置项

```yaml
cgroupDriver: systemd
```

添加完成后执行如下命令，重启kubelet服务

```terminal
systemctl daemon-reload kubelet
systemctl restart kubelet
```

若不进行上述配置，则无法正确启动infra容器

第一个控制节点添加完成后，输出中将出现如下内容：

```terminal
kubeadm join 10.10.197.101:6443 --token 7qk5p7.189wx0j9cc7mwrzi     --discovery-token-ca-cert-hash sha256:b4cfa27844f0c933922efd73f1d6f0c5e37869f48fdfd7678477d0d82154a195     --control-plane --certificate-key af535374c025dd0c5e5ca034237ad8e8b933a73a3326814d39eafc627505e3c3
```

剩下两个control plane使用上述命令添加至control plane中

#### Add The second control plane

在第二个control plane中运行上述命令，注意同样需要配置kubelet。

添加完成后，运行`realServer.service`服务

```terminal
systemctl start realServer
```

> 关于为何需要在添加完成后再将VIP配置到lo上，是因为在LVS的DR模式下，若提前配置，在节点添加过程中，节点向集群API server地址也就是VIP请求集群信息时，请求将直接发送至lo口上，有本机处理，而此时本机上并未有apiserver服务，这将导致由于无法获取集群信息引起的网络超时错误。

#### Add the third control plane

同上步骤运行

#### 查看负载均衡器运行状态

```terminal
ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.10.197.101:6443 wrr
  -> 10.10.197.97:6443            Route   10     0          0
  -> 10.10.197.98:6443            Route   10     0          0
  -> 10.10.197.99:6443            Route   10     0          0
```

可发现三个control plane已加入负载均衡器集群中正常工作。

#### 查看集群节点信息

将配置文件/etc/kubernetes/admin.conf复制到客户机的$HOME/.kube/config中，输入如下命令

```terminal
kubectl get nodes -o wide
NAME           STATUS   ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION      CONTAINER-RUNTIME
master-arm-1   Ready    master   4h53m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
master-arm-2   Ready    master   4h45m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
master-arm-3   Ready    master   4h21m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
```

### Test HA cluster

#### 关闭master-arm-1

```terminal
shutdown -h now
```

查看keepalived状态

```terminal
systemctl status keepalived
......
May 15 16:39:26 master-x86-1 Keepalived_healthcheckers[175178]: TCP_CHECK on service [10.10.197.98]:tcp:6443 failed after 3 retries.
May 15 16:39:26 master-x86-1 Keepalived_healthcheckers[175178]: Removing service [10.10.197.98]:tcp:6443 to VS [10.10.197.101]:tcp:6443
```

发现由于master-arm-1无响应，已被移除服务

查看ipvs状态

```terminal
ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.10.197.101:6443 wrr
  -> 10.10.197.97:6443            Route   10     0          1
  -> 10.10.197.99:6443            Route   10     0          0
```

负载均衡器集群中已无master-arm-1节点

查看kubernetes集群状态

```terminal
NAME           STATUS     ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION      CONTAINER-RUNTIME
master-arm-1   NotReady   master   4h57m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
master-arm-2   Ready      master   4h50m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
master-arm-3   Ready      master   4h26m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
```

master-arm-1已处于`NoReady`状态，但apiserver仍在继续服务

查看kubernetes的kube-system namespace中的pod状态

```terminal
kubectl get pods -A -o wide
AMESPACE     NAME                                   READY   STATUS        RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   coredns-7bd8c79f56-8fxzf               1/1     Terminating   2          7h31m   10.88.0.4       master-arm-1   <none>           <none>
kube-system   coredns-7bd8c79f56-b4qmm               1/1     Terminating   1          7h31m   10.88.0.5       master-arm-1   <none>           <none>
kube-system   coredns-7bd8c79f56-cdpsf               1/1     Running       0          3m49s   10.88.0.3       master-arm-2   <none>           <none>
kube-system   coredns-7bd8c79f56-tvsc7               1/1     Running       0          3m49s   10.88.0.3       master-arm-3   <none>           <none>
kube-system   etcd-master-arm-1                      1/1     Running       1          7h31m   10.10.197.101   master-arm-1   <none>           <none>
kube-system   etcd-master-arm-2                      1/1     Running       0          7h23m   10.10.197.99    master-arm-2   <none>           <none>
kube-system   etcd-master-arm-3                      1/1     Running       1          7h      10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-apiserver-master-arm-1            1/1     Running       1          7h31m   10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-apiserver-master-arm-2            1/1     Running       0          7h23m   10.10.197.99    master-arm-2   <none>           <none>
kube-system   kube-apiserver-master-arm-3            1/1     Running       1          7h      10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-controller-manager-master-arm-1   1/1     Running       2          7h31m   10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-controller-manager-master-arm-2   1/1     Running       1          7h23m   10.10.197.101   master-arm-2   <none>           <none>
kube-system   kube-controller-manager-master-arm-3   1/1     Running       1          7h      10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-proxy-mtlrm                       1/1     Running       1          7h31m   10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-proxy-mx7jt                       1/1     Running       1          7h      10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-proxy-qbd8q                       1/1     Running       0          7h24m   10.10.197.99    master-arm-2   <none>           <none>
kube-system   kube-scheduler-master-arm-1            1/1     Running       2          7h31m   10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-scheduler-master-arm-2            1/1     Running       0          7h23m   10.10.197.99    master-arm-2   <none>           <none>
kube-system   kube-scheduler-master-arm-3            1/1     Running       1          7h      10.10.197.101   master-arm-3   <none>           <none>
```

发现coredns开始在其他控制节点上重启，探测故障的时间与`liveness probe`相关，处于Terminating状态的`Pods`将一直处于该状态，因为关掉节点的kubelet已无法服务将该`Pods`删除。

coredns是一个`kubernetes`的`deployments`控制器，其中容器失效探测时间默认定义如下

```terminal
Liveness:     http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
```

即每10s探测一次，5s超时后认为一次失败，连续5次失败即失效，所以总共所需时间75S

#### 恢复master-arm-1

查看keepalived状态

```terminal
systemctl status keepalived
......
May 15 16:44:18 master-x86-1 Keepalived_healthcheckers[175178]: TCP connection to [10.10.197.98]:tcp:6443 success.
May 15 16:44:18 master-x86-1 Keepalived_healthcheckers[175178]: Adding service [10.10.197.98]:tcp:6443 to VS [10.10.197.101]:tcp:6443
```

master-arm-1恢复响应，重新加入服务

查看ipvs状态

```terminal
ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.10.197.101:6443 wrr
  -> 10.10.197.97:6443            Route   10     0          0
  -> 10.10.197.98:6443            Route   10     0          0
  -> 10.10.197.99:6443            Route   10     0          0
```

负载均衡器集群中出现master-arm-1节点

查看kubernetes集群状态

```terminal
NAME           STATUS     ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION      CONTAINER-RUNTIME
master-arm-1   Ready   master   4h57m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
master-arm-2   Ready      master   4h50m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
master-arm-3   Ready      master   4h26m   v1.18.2   10.10.197.101   <none>        Ubuntu 19.10   5.3.0-1023-raspi2   cri-o://1.18.0
```

master-arm-1重新处于`Ready`状态

查看kubernetes Pods状态

```terminal
kubectl get pods -A -o -wide
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   coredns-7bd8c79f56-cdpsf               1/1     Running   0          42m     10.88.0.3       master-arm-2   <none>           <none>
kube-system   coredns-7bd8c79f56-tvsc7               1/1     Running   0          42m     10.88.0.3       master-arm-3   <none>           <none>
kube-system   etcd-master-arm-1                      1/1     Running   2          8h      10.10.197.101   master-arm-1   <none>           <none>
kube-system   etcd-master-arm-2                      1/1     Running   0          8h      10.10.197.99    master-arm-2   <none>           <none>
kube-system   etcd-master-arm-3                      1/1     Running   1          7h38m   10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-apiserver-master-arm-1            1/1     Running   4          8h      10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-apiserver-master-arm-2            1/1     Running   0          8h      10.10.197.99    master-arm-2   <none>           <none>
kube-system   kube-apiserver-master-arm-3            1/1     Running   1          7h38m   10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-controller-manager-master-arm-1   1/1     Running   3          8h      10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-controller-manager-master-arm-2   1/1     Running   1          8h      10.10.197.101   master-arm-2   <none>           <none>
kube-system   kube-controller-manager-master-arm-3   1/1     Running   1          7h38m   10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-proxy-mtlrm                       1/1     Running   2          8h      10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-proxy-mx7jt                       1/1     Running   1          7h38m   10.10.197.101   master-arm-3   <none>           <none>
kube-system   kube-proxy-qbd8q                       1/1     Running   0          8h      10.10.197.99    master-arm-2   <none>           <none>
kube-system   kube-scheduler-master-arm-1            1/1     Running   3          8h      10.10.197.101   master-arm-1   <none>           <none>
kube-system   kube-scheduler-master-arm-2            1/1     Running   0          8h      10.10.197.99    master-arm-2   <none>           <none>
kube-system   kube-scheduler-master-arm-3            1/1     Running   1          7h38m   10.10.197.101   master-arm-3   <none>           <none>
```

可发现两个处于`Terminating`状态的Pods已被删除
