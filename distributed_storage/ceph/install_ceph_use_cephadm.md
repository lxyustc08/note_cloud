- [Install Ceph Use Cephadm](#install-ceph-use-cephadm)
    - [Environment](#environment)
  - [Procedure](#procedure)
    - [INSTALL CEPHADM](#install-cephadm)
    - [BOOTSTRAP A NEW CLUSTER](#bootstrap-a-new-cluster)
      - [USE CEPH CLI](#use-ceph-cli)
    - [ADD HOSTS TO THE CLUSTER](#add-hosts-to-the-cluster)
    - [DEPLOY ADDTIONAL MONITORS](#deploy-addtional-monitors)
      - [Set monitor numbers](#set-monitor-numbers)
      - [Use node label](#use-node-label)
      - [Manual depoly mon](#manual-depoly-mon)
    - [DEPLOY OSDS](#deploy-osds)
    - [DEPLOY ADDTIONAL MGR](#deploy-addtional-mgr)
      - [USE CEPH DASHBOARD](#use-ceph-dashboard)
  - [CEPH ORCH USAGE](#ceph-orch-usage)

# Install Ceph Use Cephadm

Cephadm通过使用容器技术与systemd，实现ceph集群的部署，在octopus版本后，ceph-depoly工具不再被官方支持，cephadm成为官方指定的部署工具，Cephadm具有下述特性

+ 仅支持Octopus版本以及其后版本的部署
+ cephadm完全使用新的orchestration API并支持全新的CLI与dashboard特性
+ cephadm使用podman或docker，因此需要节点上部署podman或docker

Cephadm使用的容器镜像为`docker.io/ceph/ceph:v15`，本次部署过程中已将其推送到本地镜像库`lxyustc.registrydomain.com:5000/ceph/ceph:v15`

ceph orch为cephadm前端，

### Environment

本次部署基本环境如下：

| hostname | IP | OS | lvm configure |
|:---:|:---:|:---:|:---:|
|worker-amd64-gpuceph-node1|10.10.197.200|Ubuntu 20.04 Server|ubuntu-vg/ceph-volume-1 100G </br> ubuntu-vg/ceph-volume-2 100G </br> ubuntu-vg/ceph-volume-3 100G </br> ubuntu-vg/ceph-volume-4 100G </br> ubuntu-vg/ceph-volume-5 100G </br> ubuntu-vg/ceph-volume-6 100G |
|worker-amd64-gpuceph-node2|10.10.197.201|Ubuntu 20.04 Server|ubuntu-vg/ceph-volume-1 100G </br> ubuntu-vg/ceph-volume-2 100G </br> ubuntu-vg/ceph-volume-3 100G </br> ubuntu-vg/ceph-volume-4 100G </br> ubuntu-vg/ceph-volume-5 100G </br> ubuntu-vg/ceph-volume-6 100G |
|worker-amd64-gpuceph-node3|10.10.197.202|Ubuntu 20.04 Server|ubuntu-vg/ceph-volume-1 100G </br> ubuntu-vg/ceph-volume-2 100G </br> ubuntu-vg/ceph-volume-3 100G </br> ubuntu-vg/ceph-volume-4 100G </br> ubuntu-vg/ceph-volume-5 100G </br> ubuntu-vg/ceph-volume-6 100G |
|woker-amd64-1|10.10.197.95|Ubuntu 20.04 Server|ubuntu-vg/ceph-volume-1 40G</br>ubuntu-vg/ceph-volume-2 40G</br>ubuntu-vg/ceph-volume-3 40G|

## Procedure

### INSTALL CEPHADM

1. 获取cephadm执行脚本，在此步骤时即可使用cephadm，**cephadm需要使用root权限运行**
   
   ```
   $ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
   $ sudo chmod +x cephadm
   ```

2. （Ubuntu系统下不建议）安装cephadm，在Ubuntu 20.04系统下，cephadm add-repo会添加cephadm仓库的gpg认证密钥，此时会扰乱ubuntu仓库正常的密钥使用
   
   ```
   $ sudo ./cephadm add-repo --release octopus
   $ ./cephadm install
   ```

### BOOTSTRAP A NEW CLUSTER

1. 在woker-amd64-gpuceph-node1上运行命令：
   
   ```
   $ sudo mkdir -p /etc/ceph
   $ sudo ./cephadm --docker --image  lxyustc.registrydomain.com:5000/ceph/ceph:v15 bootstrap --mon-ip 10.10.197.200
   ```

上述命令执行如下效果：

+ 在worker-amd64-gpuceph-node1上创建monitor(`mon`)和manager daemon(`mgr`)
+ 产生访问ceph cluster的ssh key，并添加至`/root/.ssh/authorized_keys`文件中
+ 产生一个最小的访问集群的配置文件，路径为`/etc/ceph/ceph.conf`
+ 将client.admin的密钥写入`/etc/ceph/ceph.client.admin.keyring`文件下
+ 在`/etc/ceph/ceph.pub`中写入密钥的公钥，与`/root/.ssh/authorized_keys`中一致

关于cephadm bootstrap，可使用cephadm bootstrap -h进一步了解可选项。

#### USE CEPH CLI

cephadm通过使用容器，可直接在容器中使用ceph相关命令。默认情况下，位于文件夹`/etc/ceph`下的配置文件与密钥文件作为环境变量自动加载。但若在mon节点上运行cephadm shell，则容器将从mon容器中获取集群配置与密钥文件

### ADD HOSTS TO THE CLUSTER

> **该部分命令在cephadm shell中执行**

1. 分发集群访问公钥至待添加的节点，由于cephadm bootstrap使用默认配置，因此用户均为root
   
   ```
   # ssh-copy-id -f -i /etc/ceph/ceph.pub root@worker-amd64-gpuceph-node2 
   # ssh-copy-id -f -i /etc/ceph/ceph.pub root@worker-amd64-gpuceph-node3
   ```

2. 将节点添加至集群中，**在Ubuntu 20.04中，此处ceph orch host add运行时添加host IP地址，否则出现bug，无法连接目标host导致添加失败**
   
   ```
   # ceph orch host add worker-amd64-gpuceph-node2 10.10.197.201
   # ceph orch host add worker-amd64-gpuceph-node3 10.10.197.202
   # ceph orch host add worker-amd64-1 10.10.197.95
   ```

### DEPLOY ADDTIONAL MONITORS

> **该部分命令在ceph shell中执行**

基本的添加monitor的命令为ceph orch apply mon &lt;host1,host2,host3...&gt;其他几种方式也有

#### Set monitor numbers

使用如下命令可在集群中部署特定数量的monitors

```
# ceph orch apply mon <number-of-monitors>
```

#### Use node label

可通过node label方式在指定node集合上部署monitors

1. 设置node label
   
   ```
   # ceph orch host label add worker-amd64-gpuceph-node1 mon
   # ceph orch host label add worker-amd64-gpuceph-node2 mon
   # ceph orch host label add worker-amd64-gpuceph-node3 mon
   ```

2. 使用命令在具有mon label的节点上部署mon
   
   ```
   # ceph orch apply mon label:mon
   ```


#### Manual depoly mon

此处的Manual非手动在节点上部署，而是同样采用cephadm，但不使用cephadm的自动部署机制，手动指定mon的IP或子网

1. 禁用mon自动部署
   
   ```
   # ceph orch apply mon --unmanaged
   ```

2. 部署mon
   
   ```
   # ceph orch daemon add mon worker-amd64-gpuceph-node1:10.10.197.200
   # ceph orch daemon add mon worker-amd64-gpuceph-node2:10.10.196.0/23
   ```

### DEPLOY OSDS

当前版本15.2.4下cephadm部署osd时只支持无分区、无LVM配置、无挂载、无文件系统、无ceph bluestore osd、大小大于5GB的磁盘设备，因此在当前环境下，需手动配置ceph osd。已worker-amd64-cephgpu-node1为例。

> **本部分指令执行在宿主机环境而非容器中**

1. 安装ceph-osd软件包
   
   ```
   $ sudo apt install ceph-osd
   ```

2. 检查`/etc/ceph`文件夹下是否存在ceph.conf, ceph.pub, ceph.client.admin.keyring文件，若不存在，从bootstrap节点中将上述文件拷贝至当前节点
3. 在文件夹`/var/lib/ceph/bootstrap-osd`中检查bootstrap osd认证文件是否存在，若不存在执行如下命令，查找client.bootstrap-osd的认证key信息
   
   ```
   # ceph auth list 
   ...
   client.bootstrap-osd
        key: AQBUVlhfIuuwLRAAWha9ww8pgJovgrIJXFhKag==
        caps: [mon] allow profile bootstrap-osd
   ...
   ```

4. 在文件夹`/var/lib/ceph/bootstrap-osd`创建文件ceph.keyring，内容如下
   
   ```
   [client.bootstrap-osd]
   key = AQBUVlhfIuuwLRAAWha9ww8pgJovgrIJXFhKag==
   ```

5. 依次准备ceph osd
   
   ```
   # ceph-volume lvm prepare --bluestore --data ubuntu-vg/ceph-volume-1
   # ceph-volume lvm prepare --bluestore --data ubuntu-vg/ceph-volume-2
   # ceph-volume lvm prepare --bluestore --data ubuntu-vg/ceph-volume-3
   # ceph-volume lvm prepare --bluestore --data ubuntu-vg/ceph-volume-4
   # ceph-volume lvm prepare --bluestore --data ubuntu-vg/ceph-volume-5
   # ceph-volume lvm prepare --bluestore --data ubuntu-vg/ceph-volume-6
   ```

6. 查看此时ceph osd准备情况，确认osd准备完成
   
   ```
   # ceph-volume lvm list
   

   ====== osd.0 =======

     [block]       /dev/ubuntu-vg/ceph-volume-1

         block device              /dev/ubuntu-vg/ceph-volume-1
         block uuid                ZrDjBK-NnX7-RcII-510C-LBOb-Hhje-Doc0Za
         cephx lockbox secret
         cluster fsid              bc1f0b56-f252-11ea-8930-a9753a839177
         cluster name              ceph
         crush device class        None
         encrypted                 0
         osd fsid                  28ac0671-b3fe-42c9-8bd6-f871305893c0
         osd id                    0
         osdspec affinity
         type                      block
         vdo                       0
         devices                   /dev/sda3

   ====== osd.1 =======

     [block]       /dev/ubuntu-vg/ceph-volume-2

         block device              /dev/ubuntu-vg/ceph-volume-2
         block uuid                eV0cg7-TaS7-ywij-ekRU-xRq5-gQ8Y-8E5fp9
         cephx lockbox secret
         cluster fsid              bc1f0b56-f252-11ea-8930-a9753a839177
         cluster name              ceph
         crush device class        None
         encrypted                 0
         osd fsid                  875f4381-fb3d-4758-ae17-15b995183f99
         osd id                    1
         type                      block
         vdo                       0
         devices                   /dev/sda3

   ====== osd.2 =======

     [block]       /dev/ubuntu-vg/ceph-volume-3

         block device              /dev/ubuntu-vg/ceph-volume-3
         block uuid                8zLUFO-XNaY-3tjO-PxRk-TX90-hs7X-FPGC5l
         cephx lockbox secret
         cluster fsid              bc1f0b56-f252-11ea-8930-a9753a839177
         cluster name              ceph
         crush device class        None
         encrypted                 0
         osd fsid                  5bbd8c9e-c35a-4d8e-9095-3aba30fb548f
         osd id                    2
         osdspec affinity
         type                      block
         vdo                       0
         devices                   /dev/sda3

   ====== osd.3 =======

     [block]       /dev/ubuntu-vg/ceph-volume-4

         block device              /dev/ubuntu-vg/ceph-volume-4
         block uuid                QX6aaC-BqSN-i3Dg-mNVt-QTf4-WtFk-Xepp0B
         cephx lockbox secret
         cluster fsid              bc1f0b56-f252-11ea-8930-a9753a839177
         cluster name              ceph
         crush device class        None
         encrypted                 0
         osd fsid                  6801af37-47d3-4dfa-924c-c2619f261bfd
         osd id                    3
         osdspec affinity
         type                      block
         vdo                       0
         devices                   /dev/sda3

   ====== osd.4 =======

     [block]       /dev/ubuntu-vg/ceph-volume-5

         block device              /dev/ubuntu-vg/ceph-volume-5
         block uuid                p9rSO1-twZi-mfX4-obAO-Qcwm-CxJx-yoAA0F
         cephx lockbox secret
         cluster fsid              bc1f0b56-f252-11ea-8930-a9753a839177
         cluster name              ceph
         crush device class        None
         encrypted                 0
         osd fsid                  51547d14-a68d-483a-bcd2-6ab9d8c898a2
         osd id                    4
         osdspec affinity
         type                      block
         vdo                       0
         devices                   /dev/sda3

   ====== osd.5 =======

     [block]       /dev/ubuntu-vg/ceph-volume-6

         block device              /dev/ubuntu-vg/ceph-volume-6
         block uuid                rXhMWU-EWn4-kDIf-on0n-phqm-Vuf6-BOIBLr
         cephx lockbox secret
         cluster fsid              bc1f0b56-f252-11ea-8930-a9753a839177
         cluster name              ceph
         crush device class        None
         encrypted                 0
         osd fsid                  ceae899f-c948-4bf0-8c91-7305aaca45e3
         osd id                    5
         osdspec affinity
         type                      block
         vdo                       0
         devices                   /dev/sda3
   ```

7. 激活所有osd
   
   ```
   # ceph-volume lvm activate --all
   ```

8. 使用ceph osd tree查看osd添加状态
9. 将osd纳管至ceph orch中
    
   ```
   # cephadm --image lxyustc.registrydomain.com:5000/ceph/ceph:v15 adopt --style legacy --name osd.0
   # cephadm --image lxyustc.registrydomain.com:5000/ceph/ceph:v15 adopt --style legacy --name osd.1
   # cephadm --image lxyustc.registrydomain.com:5000/ceph/ceph:v15 adopt --style legacy --name osd.2
   # cephadm --image lxyustc.registrydomain.com:5000/ceph/ceph:v15 adopt --style legacy --name osd.3
   # cephadm --image lxyustc.registrydomain.com:5000/ceph/ceph:v15 adopt --style legacy --name osd.4
   # cephadm --image lxyustc.registrydomain.com:5000/ceph/ceph:v15 adopt --style legacy --name osd.5
   ```

10. 使用ceph orch ps worker-amd64-gpuceph-node1查看daemon运行状态

    ```
    # ceph orch ps worker-amd64-gpuceph-node1
    NAME                                      HOST                        STATUS         REFRESHED  AGE  VERSION  IMAGE NAME                                     IMAGE ID      CONTAINER ID
    alertmanager.worker-amd64-gpuceph-node1   worker-amd64-gpuceph-node1  running (2d)   8m ago     2d   0.20.0   prom/alertmanager:v0.20.0                      0881eb8f169f  5fe094216d49
    crash.worker-amd64-gpuceph-node1          worker-amd64-gpuceph-node1  running (2d)   8m ago     2d   15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  05f9e4769057
    grafana.worker-amd64-gpuceph-node1        worker-amd64-gpuceph-node1  running (21h)  8m ago     2d   6.6.2    ceph/ceph-grafana:latest                       87a51ecf0b1c  c0d313f3355d
    mgr.worker-amd64-gpuceph-node1.hgenvr     worker-amd64-gpuceph-node1  running (2d)   8m ago     2d   15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  d49eb9cf6aec
    mon.worker-amd64-gpuceph-node1            worker-amd64-gpuceph-node1  running (2d)   8m ago     2d   15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  cbcc8fcede79
    node-exporter.worker-amd64-gpuceph-node1  worker-amd64-gpuceph-node1  running (2d)   8m ago     2d   0.18.1   prom/node-exporter:v0.18.1                     e5a616e4b9cf  37863870ebc0
    osd.0                                     worker-amd64-gpuceph-node1  running (22h)  8m ago     21h  15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  e358d6777215
    osd.1                                     worker-amd64-gpuceph-node1  running (22h)  8m ago     21h  15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  e8892443e90d
    osd.2                                     worker-amd64-gpuceph-node1  running (22h)  8m ago     21h  15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  589abacc22ea
    osd.3                                     worker-amd64-gpuceph-node1  running (22h)  8m ago     21h  15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  13945f14e897
    osd.4                                     worker-amd64-gpuceph-node1  running (22h)  8m ago     21h  15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  22fa7df9d6c0
    osd.5                                     worker-amd64-gpuceph-node1  running (22h)  8m ago     21h  15.2.4   lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de  0c0a7a6b25c0
    ``` 

其他节点类似，此处不再重复介绍。

> **此处存在一个问题，节点上的osd准备前应手动创建lv。不然迁移时存在osd容器无法启动的问题**

### DEPLOY ADDTIONAL MGR

基于[DEPLOY ADDTIONAL MON](#deploy-addtional-monitors)步骤中添加的mon labels，本处部署额外的mgr时可使用如下命令

> **本部分命令在cephadm shell中执行**

```
# ceph orch apply mgr label:mon
```

查看mgr服务状态，确认是否启动完成

```
# ceph orch ls mgr
NAME  RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                                     IMAGE ID
mgr       3/3  6m ago     4h   label:mon  lxyustc.registrydomain.com:5000/ceph/ceph:v15  852b28cb10de
```

#### USE CEPH DASHBOARD

ceph dashboard现在成为mgr的内置组件，在启动https时，默认情况下使用8443端口，使用前使用如下命令创建用户，创建一个名称为lxyustc，密码为XXX，角色为adminitrator的用户

```
# ceph dashboard ac-user-create lxyustc 81595390045lxy administrator
```

打开浏览器，输入当前mgr激活的地址，进入登陆页面后输入用户名、密码即可使用ceph dashboard

## CEPH ORCH USAGE

从更本质上来说，cephadm作为ceph orch的后端，前端命令仍然使用ceph orch，相关指令参考此链接[ceph orch cli](https://docs.ceph.com/docs/master/mgr/orchestrator/)