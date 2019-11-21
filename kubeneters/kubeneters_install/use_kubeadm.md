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
### Step.3 创建kuberneters集群
