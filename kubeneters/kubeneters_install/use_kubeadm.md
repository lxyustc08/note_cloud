# 使用kubeadm部署kubeneters
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
   ![Alt text](k8s界面.png "k8s界面")