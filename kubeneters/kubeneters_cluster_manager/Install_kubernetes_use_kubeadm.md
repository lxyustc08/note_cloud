- [使用kubeadm部署kubeneters X86_64](#%e4%bd%bf%e7%94%a8kubeadm%e9%83%a8%e7%bd%b2kubeneters-x8664)
	- [部署环境](#%e9%83%a8%e7%bd%b2%e7%8e%af%e5%a2%83)
	- [部署步骤](#%e9%83%a8%e7%bd%b2%e6%ad%a5%e9%aa%a4)
		- [Step.1 前期环境准备工作](#step1-%e5%89%8d%e6%9c%9f%e7%8e%af%e5%a2%83%e5%87%86%e5%a4%87%e5%b7%a5%e4%bd%9c)
		- [Step.2 安装docker kubeadm kubectl kubelet](#step2-%e5%ae%89%e8%a3%85docker-kubeadm-kubectl-kubelet)
		- [Step.3 创建kuberneters集群控制节点control plane](#step3-%e5%88%9b%e5%bb%bakuberneters%e9%9b%86%e7%be%a4%e6%8e%a7%e5%88%b6%e8%8a%82%e7%82%b9control-plane)
		- [Step4. 添加工作节点](#step4-%e6%b7%bb%e5%8a%a0%e5%b7%a5%e4%bd%9c%e8%8a%82%e7%82%b9)
		- [Step 5. 安装dashboard](#step-5-%e5%ae%89%e8%a3%85dashboard)
		- [Step 6. 访问dashboard](#step-6-%e8%ae%bf%e9%97%aedashboard)
- [使用kubeadm部署Kubernetes ARM64 v8](#%e4%bd%bf%e7%94%a8kubeadm%e9%83%a8%e7%bd%b2kubernetes-arm64-v8)
	- [基础设施](#%e5%9f%ba%e7%a1%80%e8%ae%be%e6%96%bd)
	- [容器运行环境安装](#%e5%ae%b9%e5%99%a8%e8%bf%90%e8%a1%8c%e7%8e%af%e5%a2%83%e5%ae%89%e8%a3%85)
	- [安装kubeadm，kubectl，kubelet](#%e5%ae%89%e8%a3%85kubeadmkubectlkubelet)
	- [构建Kubernetes控制平面](#%e6%9e%84%e5%bb%bakubernetes%e6%8e%a7%e5%88%b6%e5%b9%b3%e9%9d%a2)
		- [与AMD64架构区别](#%e4%b8%8eamd64%e6%9e%b6%e6%9e%84%e5%8c%ba%e5%88%ab)
		- [构建步骤](#%e6%9e%84%e5%bb%ba%e6%ad%a5%e9%aa%a4)

# 使用kubeadm部署kubeneters X86_64
## 部署环境
使用虚拟机部署
1. 宿主机环境　　
   
<table>
    <tr>
        <th>硬件配置</th>
        <th>软件配置</th>
    </tr>
    <tr>
        <td>
        CPU: Intel Core i7-8550U @ 1.80Ghz X 1 ４核8线程</br>
        内存: 16GB
        </td>
        <td>
        操作系统: Ubunt 18.04.3 LTS desktop</br>
        Hypervisor: KVM + qemu
        </td>
    </tr>
</table>　　

2. 虚拟机集群环境
<table>
    <tr>
        <th>节点名称</th>
        <th>节点配置</th>
        <th>操作系统</th>
        <th>IP地址</th>
    </tr>
    <tr>
        <td>master</td>
        <td>
        VCPU: 3</br>
        vMem: 3GB
        </td>
        <td>
        Ubuntu 18.04.3 LTS server
        </td>
        <td>
        192.168.122.25
        </td>
    </tr>
    <tr>
        <td>slave-node1</td>
        <td>
        VCPU: 2</br>
        vMem: 2GB
        </td>
        <td>
        Ubuntu 18.04.3 LTS server
        </td>
        <td>
        192.168.122.24
        </td>
    </tr>
    <tr>
        <td>slave-node2</td>
        <td>
        VCPU: 2</br>
        vMem: 2GB
        </td>
        <td>
        Ubuntu 18.04.3 LTS server
        </td>
        <td>
        192.168.122.147
        </td>
    </tr>
</table>

## 部署步骤
### Step.1 前期环境准备工作
1. 登陆集群各个节点，关闭swap  
   通过编辑/etc/fstab，注释掉swap挂载点
   ```terminal
   UUID=26062bf1-e5d3-4411-b92d-0dae540a629c / ext4 defaults 0 0
   UUID=81aeccc7-a4e8-4d62-8d28-8eb4491dae5d /boot ext4 defaults 0 0
   #/swap.img      none    swap    sw      0       0
   ```
   重启
   ```terminal
   reboot
   ```
   确认swap已经关闭
   ```terminal
   root@master:~# free -h
                 total        used        free      shared  buff/cache   available
   Mem:           3.0G        827M        432M        1.7M        1.7G        2.1G
   Swap:            0B          0B          0B
   ```
2. 修改源配置，将k8s相关工具镜像源指向国内镜像站  
   ```terminal
   vim /etc/apt/source.list.d/k8s.list
   ```
   k8s.list内容如下
   ```terminal
   deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
   ```
   添加k8s源密钥
   ```terminal
   gpg --keyserver keyserver.ubuntu.com --recv-keys BA07F4FB
   gpg --export --armor BA07F4FB | apt-key add -
   ```
3. 添加docker-ce源，源为国内镜像站  
   直接在/etc/apt/source.list中添加内容
   ```terminal
   deb https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu bionic stable
   ```
   添加docker-ce源密钥
   ```terminal
   curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
   ```
4. 更新源
   ```terminal
   apt update
   ```
   输出信息无错误即代表工作完成成功
### Step.2 安装docker kubeadm kubectl kubelet
登陆各个节点，执行下列步骤
1. 安装docker-ce，版本为18.06.2
   ```terminal
   apt install docker-ce=18.06.2~ce~3-0~ubuntu
   ```
   配置docker daemon
   ```terminal
   vim /etc/docker/daemon.json
   ```
   内容如下
   ```json
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   ```
   > 配置docker daemon的目的在于将cgroup的驱动设置为systemd  

   重启docker daemon
   ```terminal
   systemctl daemon-reload
   systemctl restart docker
   ```
2. 安装kubeadm kubectl kubelet
   ```terminal
   apt install kubectl kubelet kubeadm
   ```
### Step.3 创建kuberneters集群控制节点control plane
1. 通过命令确认需要使用的镜像
   ```terminal
   kubeadm config images list
   ```
   输出如下
   ```terminal
   k8s.gcr.io/kube-apiserver:v1.16.1
   k8s.gcr.io/kube-controller-manager:v1.16.1
   k8s.gcr.io/kube-scheduler:v1.16.1
   k8s.gcr.io/kube-proxy:v1.16.1
   k8s.gcr.io/pause:3.1
   k8s.gcr.io/etcd:3.3.15-0
   k8s.gcr.io/coredns:1.6.2
   ```
   编辑脚本k8s-images.sh，从阿里云镜像站中拉取上述镜像
   ```shell
   #!/bin/bash
   images=(
       kube-apiserver:v1.16.1
       kube-controller-manager:v1.16.1
       kube-scheduler:v1.16.1
       kube-proxy:v1.16.1
       pause:3.1
       etcd:3.3.15-0
       coredns:1.6.2
   )

   for imageName in ${images[@]} ; do
       docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
       docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
       docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
   done
   ```
   > 该脚本完成的工作为：从阿里云镜像站中拉取镜像，并将镜像标签修改为kubeadm部署时所需的样式  

   使用docker images查看镜像，输出如下：
   ```terminal
   REPOSITORY                                                                       TAG                 IMAGE ID            CREATED             SIZE
   k8s.gcr.io/kube-proxy                                                            v1.16.1             0d2430db3cd0        7 weeks ago         86.1MB
   k8s.gcr.io/kube-apiserver                                                        v1.16.1             f15aad0426f5        7 weeks ago         217MB
   k8s.gcr.io/kube-controller-manager                                               v1.16.1             ba306669806e        7 weeks ago         163MB
   k8s.gcr.io/kube-scheduler                                                        v1.16.1             e15192a92182        7 weeks ago         87.3MB
   k8s.gcr.io/etcd                                                                  3.3.15-0            b2756210eeab        2 months ago        247MB
   k8s.gcr.io/coredns                                                               1.6.2               bf261d157914        3 months ago        44.1MB
   k8s.gcr.io/pause                                                                 3.1                 da86e6ba6ca1        23 months ago       742kB
   ```
2. 使用kubeadmin创建一个单独的control plane
   ```terminal
   kubeadm init --pod-network-cidr=172.15.0.0/16
   ```
   > 本次部署的集群使用calico网络插件，集群初始化时需要指定--pod-network-cidr参数  
   > **--pod-network-cidr参数不可与现有集群节点所在的网络地址重合**
3. 使用kubectl get pods -A查看pods运行状态，除网络服务外其他pods应该为Running状态
4. 部署calico网络插件  
   下载部署文件
   ```terminal
   curl https://docs.projectcalico.org/v3.10/manifests/calico.yaml -O
   ```
   修改文件配置，将cidr配置为kubeadm init命令中的参数
   ```yaml
   ...
            - name: CALICO_IPV4POOL_CIDR
              value: "172.15.0.0/16"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
    ...
   ```
5. 应用calico部署配置
   ```terminal
   kubectl apply -f calico.yaml
   ```
   此时所有pods运行状态为Running
   ```terminal
   kubectl get pods -A
   NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE
   kube-system            calico-kube-controllers-6b64bcd855-pdlcc     1/1     Running   1          2d4h
   kube-system            calico-node-898ql                            1/1     Running   1          2d4h
   kube-system            coredns-5644d7b6d9-4955j                     1/1     Running   1          2d4h
   kube-system            coredns-5644d7b6d9-rzb2q                     1/1     Running   1          2d4h
   kube-system            etcd-master                                  1/1     Running   1          2d4h
   kube-system            kube-apiserver-master                        1/1     Running   1          2d4h
   kube-system            kube-controller-manager-master               1/1     Running   1          2d4h
   kube-system            kube-proxy-mq8jx                             1/1     Running   1          2d4h
   kube-system            kube-scheduler-master                        1/1     Running   1          2d4h
   ```
6. 设置控制节点访问权限，使之可以访问k8s集群
   ```terminal
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config
   ```
   >对于上述命令而言，若使用root，只保留前两步即可，对于普通用户权限而言，三步全部需要保留
### Step4. 添加工作节点
1. 查看集群token
   ```terminal
   # kubeadm token list
   TOKEN     TTL       EXPIRES   USAGES    DESCRIPTION   EXTRA GROUPS
   kms85w.wcu19x5th3mmiy3j   23h       2019-11-22T08:14:13Z   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
   ```
2. 获取CA证书的hash
   ```terminal
   # openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
   d3265186b687336f3ec83cd7a76a3082a2c8ceb827d3a8edc95d3704a466b132
   ```
3. 使用k8s-images.sh分别在slave-node1, slave-node2中拉取k8s镜像
4. 分别在slave-node1, slave-node2中执行命令
   ```terminal
   # kubeadm join --token s773h1.ehz56hv3xi2pl5ai 192.168.122.25:6443 --discovery-token-ca-cert-hash sha256:d3265186b687336f3ec83cd7a76a3082a2c8ceb827d3a8edc95d3704a466b132
   ```
   将节点加入到集群中
5. 在控制节点中查看节点状态
   ```terminal
   # kubectl get nodes -A
   NAME          STATUS   ROLES    AGE    VERSION
   master        Ready    master   2d5h   v1.16.1
   slave-node1   Ready    <none>   44h    v1.16.1
   slave-node2   Ready    <none>   44h    v1.16.1
   ```
   工作节点添加成功
### Step 5. 安装dashboard
1. 准备dashboard镜像
   ```terminal
   # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1
   # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
   ```
2. 下载dashboard配置文件
   ```terminal
   # wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml
   ```
3. 应用dashboard配置文件
   ```terminal
   # kubectl apply -f recommended.yaml
   ```
4. 查看dashboard相关pods
   ```terminal
   # kubectl get nodes -A
   NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE
   kubernetes-dashboard   dashboard-metrics-scraper-76585494d8-vm2tf   1/1     Running   1          38h
   kubernetes-dashboard   kubernetes-dashboard-b65488c4-99hqs          1/1     Running   1          38h
   ```
### Step 6. 访问dashboard
1. 复制集群权限信息到宿主机中
   ```terminal
   # scp /etc/kubernetes/admin.conf lxyustc@192.168.122.1:/home/lxyustc
   ```
2. 配置宿主机集群访问权限
   ```terminal
   # mkdir -p $HOME/.kube
   # chown $(id -u):$(id -g) $HOME/.kube/config
   ```
3. 开启代理
   ```terminal
   kubectl proxy
   ```
4. 配置dashboard账号，创建文件dashboard-adminuser.yaml，内容如下
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: lxyustc
     namespace: kube-system

   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: lxyustc
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: lxyustc
     namespace: kube-system
   ```
5. 应用dashboard-adminuser
   ```terminal
   kubectl apply -f dashboard-adminuser.yaml
   ```
6. 获取用户token
   ```terminal
   # kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep lxyustc | awk '{print $1}') | grep token
   Name:         lxyustc-token-qs9g5
   Type:  kubernetes.io/service-account-token
   token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkozU0pyTEROWGhDQ2VPUkRWbl82TUlnUHlJSXNzNlduU2tsaEJrMnlwWmsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJseHl1c3RjLXRva2VuLXFzOWc1Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6Imx4eXVzdGMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxODBhODcxNC03MDRhLTQ0MTMtOWQwMS1jMmE2NmJkNjFiZDUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06bHh5dXN0YyJ9.CYnQX2FCRG21WjUYOvRjkT3PtlvA4b7sI6o8mQvFcMqk0LX6DBXinWYf8vEqoxWHFtq13qD8XGe3rBE-95hkbnH08ess3C_PmPiaBq8Lx-JSXRKLDpFfRt1zldSJglLGmbqgBkaLysOVvQ7NVDxAPYAENXbOIl-HVNYWCaIkWsoeGrqR_tmoDzxyNZwQPUJRgS0QKktVlePPDRMTr0twIsAJn_duxUsSSB3P0Fi1F8eT3lz3io7EmlMGNsaZDvElqU9kumX7f83bKEtiYUw-UVbnjOjR9C45w4tLVI2G_LcBE0A9N4kRTow97aYKMyysf8JL4OL2woHFO4yPscxHOw
   ```
7. 登陆dashboard，URL: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login  
   复制6中得到的token,到token一栏中即可成功登陆  
   ![Alt text](pictures/k8s界面.png)  

# 使用kubeadm部署Kubernetes ARM64 v8

## 基础设施
使用树莓派4 集群进行部署，节点环境如下：

<table>
	<tr>
		<th>节点名称</th>
		<th>节点配置</th>
		<th>节点操作系统</th>
		<th>节点IP地址</th>
	</tr>
	<tr>
		<td>master-arm</td>
		<td>CPU：4 核 Cortex A72 </br> 内存： 4GB</td>
		<td>Ubuntu 19.10 Server</td>
		<td>10.10.197.97</td>
	</tr>
	<tr>
		<td>slave-node1-arm</td>
		<td>CPU：4 核 Cortex A72 </br> 内存： 4GB</td>
		<td>Ubuntu 19.10 Server</td>
		<td>10.10.197.98</td>
	</tr>
	<tr>
		<td>slave-node2-arm</td>
		<td>CPU：4 核 Cortex A72 </br> 内存： 4GB</td>
		<td>Ubuntu 19.10 Server</td>
		<td>10.10.197.99</td>
	</tr>
</table>

**Note:**
1. 2019年12月5日之前，Ubuntu 19.10 for Raspberry Pi 4 的内核存在Bug，对于4GB内存的树莓派型号不能识别USB设备，需要在/boot/firmware/usercfg.txt中配置参数`total_mem=3072`, 该bug目前已经修复，修复后的内核版本为`5.3.0-1014-raspi2`；
2. Ubuntu 19.10 for Raspberry Pi 4的cgroup子系统中，默认情况下并未开启memory子系统，需要通过内核`cmd`参数开启，开启如下：
   + Ubuntu 19.10 Server系统的内核`cmd`配置通过`/boot/firmware/nobtcmd.txt`中指定，在配置文件`/boot/firmware/nobtcmd.txt`后加入`cgroup_enable=memory cgroup_memory=1`，保存后重启；
   + 重启后使用命令`lssubsys`
     	```terminal
     	cpuset
       cpu,cpuacct
       blkio
       memory
       devices
       freezer
       net_cls,net_prio
       perf_event
       pids
       rdma
		```
		确认已开启memory cgroup子系统

## 容器运行环境安装
本次直接使用Ubuntu 19.10 for Raspberry Pi 4附带的docker软件包安装
1. 配置镜像源，与AMD64架构类似，不过使用的是ubuntu-ports源，文件`/etc/apt/source.list`内容如下：
```terminal
## Note, this file is written by cloud-init on first boot of an instance
## modifications made here will not survive a re-bundle.
## if you wish to make changes you can:
## a.) add 'apt_preserve_sources_list: true' to /etc/cloud/cloud.cfg
##     or do the same in user-data
## b.) add sources in /etc/apt/sources.list.d
## c.) make changes to template file /etc/cloud/templates/sources.list.tmpl

# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan main restricted
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan main restricted

## Major bug fix updates produced after the final release of the
## distribution.
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-updates main restricted
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-updates main restricted

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team. Also, please note that software in universe WILL NOT receive any
## review or updates from the Ubuntu security team.
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan universe
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-updates universe
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-updates universe

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team, and may not be under a free licence. Please satisfy yourself as to
## your rights to use the software. Also, please note that software in
## multiverse WILL NOT receive any review or updates from the Ubuntu
## security team.
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan multiverse
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-updates multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-updates multiverse

## N.B. software from this repository may not have been tested as
## extensively as that contained in the main release, although it includes
## newer versions of some applications which may provide useful features.
## Also, please note that software in backports WILL NOT receive any review
## or updates from the Ubuntu security team.
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-backports main restricted universe multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-backports main restricted universe multiverse

## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu eoan partner
# deb-src http://archive.canonical.com/ubuntu eoan partner

deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-security main restricted
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-security main restricted
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-security universe
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-security universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-security multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports eoan-security multiverse
```
2. 使用如下命令安装docker
```terminal
# apt update
# apt install docker.io
```
3. 配置/etc/docker/daemon.json  
   配置内容如下：
   ```json
   {
           "exec-opts": ["native.cgroupdriver=systemd"],
           "log-driver": "json-file",
           "log-opts": {
                        "max-size": "100m"
           },
           "storage-driver": "overlay2",
           "registry-mirrors": ["https://dockerhub.azk8s.cn"]
   }
   ```  
   该配置配置镜像加速器，cgroupdriver等参数，与AMD64架构类似；
4. 重启docker
   ```terminal
   # systemctl daemond-reload
   # systemctl restart docker
   ```

## 安装kubeadm，kubectl，kubelet
1. 配置Kubernetes工具源，配置文件`/etc/apt/source.list.d/k8s.list`如下：
   ```terminal
   deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
   ```
2. 添加软件源key
   ```terminal
   curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
   ```
3. 安装kubeadm, kubectl, kubelet
   ```terminal
   # apt update
   # apt install kubeadm kubectl kubelet
   ```

## 构建Kubernetes控制平面
### 与AMD64架构区别
1. 与AMD64架构不同，arm 64 v8架构部署Kubernetes控制平面时使用的镜像需为arm64架构镜像，国内阿里云镜像站中并未同步arm64架构容器镜像，在此处使用微软Azure建立的[GCR镜像站](http://mirror.azk8s.cn/help/gcr-proxy-cache.html)作为加速。
2. calino网络插件不支持Arm 64架构，只能使用flannel插件，在kubeadm init时需要注意`--pod-network-cidr`参数配置
3. dashboard使用的镜像`kubernetesui/metrics-scraper`从1.0.2版本开始才支持Arm 64架构，因此对象描述文件中的`kubernetesui/metrics-scraper`版本需>=1.0.2；

### 构建步骤
1. 创建control plane  
   1.16版本的kubeadm init增加了`--image-repository`参数，该参数可指定容器镜像仓库，使用Azure GCR时命令如下：
   ```terminal
   # kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository="gcr.azk8s.cn/google_containers"
   ```
   此处的`--pod-network-cidr`必须为10.244.0.0/16，为flannel网络插件必须配置的，且不可改变；
2. 参照AMD64架构下kubeconfig配置文件配置，授予master节点集群访问权限，等待除core dns pod外的pod创建完成，如下所示：
   ```terminal
   NAME                                 READY   STATUS    RESTARTS   AGE
   coredns-667f964f9b-2n5wh             1/1     Pending   0          
   coredns-667f964f9b-lf6sq             1/1     Pending   0          
   etcd-master-arm                      1/1     Running   0          
   kube-apiserver-master-arm            1/1     Running   0          
   kube-controller-manager-master-arm   1/1     Running   0          
   kube-scheduler-master-arm            1/1     Running   0          
   ```
3. 安装flannel插件
   ```terminal
   # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
   ```
   等待coredns状态变为Running
4. 添加Node  
   与AMD64，通过token将节点添加至集群中
   + slave-node1-arm
     ```terminal
     kubeadm join --token qou0kk.ir14bq6dpa2r6w79 10.10.197.97:6443 --discovery-token-ca-cert-hash sha256:7ae1c47e0fcec181ef32e3736f15465f6529f7154349ca31380e589c429fa91d
     ```
   + slave-node2-arm
     ```terminal
	 kubeadm join --token qou0kk.ir14bq6dpa2r6w79 10.10.197.97:6443 --discovery-token-ca-cert-hash sha256:7ae1c47e0fcec181ef32e3736f15465f6529f7154349ca31380e589c429fa91d
	 ```
   在master中查看节点运行状态
   ```terminal
   # kubectl get nodes
   NAME              STATUS   ROLES    AGE   VERSION
   master-arm        Ready    master   31h   v1.16.3
   slave-node1-arm   Ready    <none>   30h   v1.16.3
   slave-node2-arm   Ready    <none>   30h   v1.16.3
   ```
5. 测试，编写test.yaml，内容如下：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
           name: nginx
   spec:
           selector:
                   matchLabels:
                           app: nginx
           replicas: 4
           template:
                   metadata:
                           labels:
                                   app: nginx
                   spec:
                           containers:
                                   - name: nginx
                                     image: arm64v8/nginx:1.16
                                     ports:
                                             - containerPort: 80
   ```
   部署该对象
   ```terminal
   # kubectl apply -f test.yaml
   ```
   查看对象
   ```terminal
   # kubectl describe deployment nginx
   Name:                   nginx
   Namespace:              default
   CreationTimestamp:      Fri, 06 Dec 2019 18:24:52 +0800
   Labels:                 <none>
   Annotations:            deployment.kubernetes.io/revision: 2
                           kubectl.kubernetes.io/last-applied-configuration:
                             {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"replicas":4,"selec...
   Selector:               app=nginx
   Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
   StrategyType:           RollingUpdate
   MinReadySeconds:        0
   RollingUpdateStrategy:  25% max unavailable, 25% max surge
   ```
6. 安装dashboard  
   获取dashboard对象配置文件
   ```terminal
   # wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml
   ```
   修改`kubernetesui/metrics-scraper`版本为1.0.1，创建dashboard对象
   ```terminal
   # kubectl apply -f recommended.yaml
   ```
   查看kubernetes-dashboard部署
   ```terminal
   # kubectl get deployments -n kubernetes-dashboard
   NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
   dashboard-metrics-scraper   1/1     1            1           22h
   kubernetes-dashboard        1/1     1            1           22h
   ```
7. 类似于AMD64架构，将Kubernetes集群的配置文件复制到想要从集群外节点访问集群的节点中  
   在远端节点创建文件夹
   ```terminal
   # mkdir kubernetes_arm_conf
   # scp root@master_ip:/etc/kubernetes/admin.conf
   ```
8. 在master节点上创建用户文件admin.yaml
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: lxyustc
     namespace: kube-system

   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: lxyustc
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: lxyustc
     namespace: kube-system
   ```
9. 使用kubeclt应用该文件
    ```terminal
	# kubectl apply -f admin.yaml
	```
10. 获取用户token
    ```terminal
	# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep lxyustc | awk '{print $1}') | grep token
	``` 
11. 远端节点中运行proxy
   ```terminal
   # kubectl --kubeconfig admin.conf proxy
   ```
   登陆dashboard，URL: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login  
   填入10中获得的token  
   ![Alt Text](./pictures/k8s_arm界面.png)

# Web UI 部署

无论是X86还是ARM部署方式类似，直接可以通过配置yaml文件进行部署。

### step.1 编辑配置文件
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.0.0-rc6
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.4
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}

```

注意点如下：

1. dashboard service默认情况下为`ClusterIP`，此时需通过kubectl proxy代理方式进行访问；
2. 将dashboard service类型修改为`NodePord`或`LoadBalancer`，则可通过工作节点IP或外部IP访问；

### step.2 编辑serviceaccount

正如kubernetes认证相关描述，kubernetes用户分为两种normal user与service account，使用service account是访问dashboard较简单的方式，典型的serviceaccount及其与rbac授权模组绑定的配置文件如下：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
    name: lxyustc
    namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
    name: lxyustc-rolebinding
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
subjects:
- kind: ServiceAccount
  name: lxyustc
  namespace: default
```

该配置文件创建了一个名称为lxyustc的serviceaccount，该serviceaccount与`admin`角色绑定。

注意：

1. 复制token时可能存在格式问题，导致认证无效，建议通过记事本洗掉格式，并复制为纯文本
