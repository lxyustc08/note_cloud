- [Install Ceph](#install-ceph)
  - [Important Things](#important-things)
  - [Environment](#environment)
    - [deploy node](#deploy-node)
    - [monitor node](#monitor-node)
    - [osd node](#osd-node)
  - [Procedure](#procedure)
    - [PREFLIGHT CHECKLIST](#preflight-checklist)
      - [set up ntp service on each ceph node](#set-up-ntp-service-on-each-ceph-node)
      - [create user on ceph node for install ceph](#create-user-on-ceph-node-for-install-ceph)
      - [config less passwd ssh for deploy node](#config-less-passwd-ssh-for-deploy-node)
    - [CREATE A CLUSTER](#create-a-cluster)
      - [Inital a single monitor node](#inital-a-single-monitor-node)
      - [ADD more monitors](#add-more-monitors)
      - [ADD RGW INSTANCE](#add-rgw-instance)
    - [TEST](#test)

# Install Ceph

## Important Things

**ceph使用hostname作为节点名，在deploy node上配置hosts时需要与各个节点的hostname保持一致**

## Environment

***

### deploy node

|IP|hostname|arch|OS|
|:---:|:---:|:---:|:---:|
|10.10.197.96|master-x86-1|amd64|ubuntu 20.04 LTS|

deploy node上部署ceph-deploy工具，对于python 3.8以上的版本，由于platform.linux_distribution函数被废除，bug详见[此链接](https://github.com/ceph/ceph-deploy/pull/496)，导致ubuntu deb包中的包使用时报错，因此直接使用git clone源码进行安装。

**强力建议使用此种方式安装ceph-deploy，避免ceph-deploy deb包对后续安装部署过程造成影响**

1. 下载源码
   
   ```
   git clone https://github.com/ceph/ceph-deploy.git 
   ```

2. 进入源码目录
   
   ```
   ./bootstrap 3
   ```

3. 在`/usr/bin`下创建软链接
   
   ```
   sudo ln -s /home/lxyustc/ceph-deploy/ceph-deploy ceph-deploy
   ```


### monitor node

|IP|hostname|arch|OS|
|:---:|:---:|:---:|:---:|
|10.10.197.200|worker-amd64-gpuceph-node1|amd64|ubuntu 20.04 LTS|
|10.10.197.201|worker-amd64-gpuceph-node2|amd64|ubuntu 20.04 LTS|
|10.10.197.202|worker-amd64-gpuceph-node3|amd64|ubuntu 20.04 LTS|

### osd node 

|IP|hostname|arch|OS|partition|
|:---:|:---:|:---:|:---:|:---:|
|10.10.197.200|worker-amd64-gpuceph-node1|amd64|ubuntu 20.04 LTS|ubuntu-vg/ceph-volume-1 100GB <br> ubuntu-vg/ceph-volume-2 100GB <br> ubuntu-vg/ceph-volume-3 100GB <br> ubuntu-vg/ceph-volume-4 100GB <br> ubuntu-vg/ceph-volume-5 100GB <br> ubuntu-vg/ceph-volume-6 100GB|
|10.10.197.201|worker-amd64-gpuceph-node2|amd64|ubuntu 20.04 LTS|ubuntu-vg/ceph-volume-1 100GB <br> ubuntu-vg/ceph-volume-2 100GB <br> ubuntu-vg/ceph-volume-3 100GB <br> ubuntu-vg/ceph-volume-4 100GB <br> ubuntu-vg/ceph-volume-5 100GB <br> ubuntu-vg/ceph-volume-6 100GB|
|10.10.197.202|worker-amd64-gpuceph-node3|amd64|ubuntu 20.04 LTS|ubuntu-vg/ceph-volume-1 100GB <br> ubuntu-vg/ceph-volume-2 100GB <br> ubuntu-vg/ceph-volume-3 100GB <br> ubuntu-vg/ceph-volume-4 100GB <br> ubuntu-vg/ceph-volume-5 100GB <br> ubuntu-vg/ceph-volume-6 100GB|

## Procedure

***

### PREFLIGHT CHECKLIST

#### set up ntp service on each ceph node

1. 安装ntp
   
   ```
   # apt install ntpsec
   ```

2. 配置ntp服务，保证所有ceph node上的ntp指向同一个ntp服务器，本处使用默认ntp配置即可，更多的ntp配置信息参考[如下链接](http://www.ntp.org/)。本次配置内容如下
   
   ```confg
   # /etc/ntpsec/ntp.conf, configuration for ntpd; see ntp.conf(5) for help

   driftfile /var/lib/ntpsec/ntp.drift
   leapfile /usr/share/zoneinfo/leap-seconds.list

   # To enable Network Time Security support as a server, obtain a certificate
   # (e.g. with Let's Encrypt), configure the paths below, and uncomment:
   # nts cert CERT_FILE
   # nts key KEY_FILE
   # nts enable mintls TLS1.3

   # You must create /var/log/ntpsec (owned by ntpsec:ntpsec) to enable logging.
   #statsdir /var/log/ntpsec/
   #statistics loopstats peerstats clockstats
   #filegen loopstats file loopstats type day enable
   #filegen peerstats file peerstats type day enable
   #filegen clockstats file clockstats type day enable

   # This should be maxclock 7, but the pool entries count towards maxclock.
   tos maxclock 11

   # Comment this out if you have a refclock and want it to be able to discipline
   # the clock by itself (e.g. if the system is not connected to the network).
   tos minclock 4 minsane 3

   # Specify one or more NTP servers.


   # Public NTP servers supporting Network Time Security:
   # server time.cloudflare.com:1234 nts

   # Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
   # on 2011-02-08 (LP: #104525). See https://www.pool.ntp.org/join.html for
   # more information.
   pool 0.ubuntu.pool.ntp.org iburst
   pool 1.ubuntu.pool.ntp.org iburst
   pool 2.ubuntu.pool.ntp.org iburst
   pool 3.ubuntu.pool.ntp.org iburst

   # Use Ubuntu's ntp server as a fallback.
   server ntp.ubuntu.com

   # Access control configuration; see /usr/share/doc/ntpsec-doc/html/accopt.html
   # for details.
   #
   # Note that "restrict" applies to both servers and clients, so a configuration
   # that might be intended to block requests from certain clients could also end
   # up blocking replies from your own upstream servers.

   # By default, exchange time with everybody, but don't allow configuration.
   restrict default kod nomodify nopeer noquery limited

   # Local users may interrogate the ntp server more closely.
   restrict 127.0.0.1
   restrict ::1
   ```

#### create user on ceph node for install ceph

从安全角度考虑，在各个ceph node上使用额外的user账号，而非root账号，在各个ceph node上创建用户：

1. 创建用户，名称为ceph-kubernetes
   
   ```
   useradd -d /home/ceph-kubernetes -m ceph-kubernetes
   passwd ceph-kubernetes
   ```

2. 为ceph-kubernetes添加sudo权限
   
   ```
   echo "ceph-kubernetes ALL = (root) NOPASSWD:ALL" | tee /etc/sudoers.d/ceph-kubernetes
   chmod 0440 /etc/sudoers.d/ceph-kubernetes
   ```

#### config less passwd ssh for deploy node

在deploy node上配置对ceph node的免密访问

1. 生成密钥，使用非root，非sudo命令
   
   ```
   ssh-key-gen
   ```

2. 分发密钥
   
   ```shell
   for i in 95 97 98 99
   do
        ssh-copy-id ceph-kubernetes@10.10.197.${i}
   done
   ```

3. 配置deploy node ssh客户端，`~/.ssh/config`
   
   ```confg
   Host worker-amd64-1
           Hostname 10.10.197.95
           User ceph-kubernetes
   Host master-arm-1
           Hostname 10.10.197.98
           User ceph-kubernetes
   Host master-arm-2
           Hostname 10.10.197.99
           User ceph-kubernetes
   Host master-arm-3
           Hostname 10.10.197.97
           User ceph-kubernetes
   ```

### CREATE A CLUSTER

配置deploy node的`/etc/hosts`

```
10.10.197.95 worker-amd64-1
10.10.197.98 master-arm-1
10.10.197.99 master-arm-2
10.10.197.97 master-arm-3
```

创建集群之前，在deploy node上创建集群配置文件夹`~/ceph-kubernetes-config`

```
mkdir ceph-kubernetes-config
```

在上述文件夹中使用ceph-deploy工具

#### Inital a single monitor node

1. 初始化集群配置，在`~/ceph-kubernetes-config`下运行命令
   
   ```
   ceph-deploy new worker-amd64-gpuceph-node1
   [ceph_deploy.conf][DEBUG ] found configuration file at: /home/lxyustc/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (2.0.2): /usr/bin/ceph-deploy new monitor-node1
   [ceph_deploy.cli][INFO  ] ceph-deploy options:
   [ceph_deploy.cli][INFO  ]  verbose                       : False
   [ceph_deploy.cli][INFO  ]  quiet                         : False
   [ceph_deploy.cli][INFO  ]  username                      : None
   [ceph_deploy.cli][INFO  ]  overwrite_conf                : False
   [ceph_deploy.cli][INFO  ]  ceph_conf                     : None
   [ceph_deploy.cli][INFO  ]  cluster                       : ceph
   [ceph_deploy.cli][INFO  ]  mon                           : ['monitor-node1']
   [ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
   [ceph_deploy.cli][INFO  ]  fsid                          : None
   [ceph_deploy.cli][INFO  ]  cluster_network               : None
   [ceph_deploy.cli][INFO  ]  public_network                : None
   [ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf object at 0x7f6dd2f29c40>
   [ceph_deploy.cli][INFO  ]  default_release               : False
   [ceph_deploy.cli][INFO  ]  func                          : <function new at 0x7f6dd2eb11f0>
   [ceph_deploy.new][DEBUG ] Creating new cluster named ceph
   [ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
   [monitor-node1][DEBUG ] connected to host: master-x86-1
   [monitor-node1][INFO  ] Running command: ssh -CT -o BatchMode=yes monitor-node1 true
   [monitor-node1][DEBUG ] connection detected need for sudo
   [monitor-node1][DEBUG ] connected to host: monitor-node1
   [monitor-node1][INFO  ] Running command: sudo /bin/ip link show
   [monitor-node1][INFO  ] Running command: sudo /bin/ip addr show
   [monitor-node1][DEBUG ] IP addresses found: ['10.10.197.101', '10.88.0.0', '10.10.197.98']
   [ceph_deploy.new][DEBUG ] Resolving host monitor-node1
   [ceph_deploy.new][DEBUG ] Monitor monitor-node1 at 10.10.197.98
   [ceph_deploy.new][DEBUG ] Monitor initial members are ['monitor-node1']
   [ceph_deploy.new][DEBUG ] Monitor addrs are ['10.10.197.98']
   [ceph_deploy.new][DEBUG ] Creating a random mon key...
   [ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
   [ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...
   ```

2. 编辑集群配置文件`ceph.conf`，设置public network属性值，通过该属性值指定使用的网络接口
   
   ```
   public network = 10.10.196.0/23
   ```

3. 安装ceph packages，先安装osd-node1与monitor-node1，后续节点再进行添加
   
   ```
   ceph-deploy install worker-amd64-gpuceph-node1 worker-amd64-gpuceph-node2 worker-amd64-gpuceph-node3
   ```

4. 部署初始的monitor，并获取相关的keyrings
   
   ```
   ceph-deploy mon create-initial

   ......
   [master-arm-1][DEBUG ] ********************************************************************************
   [master-arm-1][INFO  ] monitor: mon.master-arm-1 is running
   [master-arm-1][INFO  ] Running command: sudo ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.master-arm-1.asok mon_status
   [ceph_deploy.mon][INFO  ] processing monitor mon.master-arm-1
   [master-arm-1][DEBUG ] connection detected need for sudo
   [master-arm-1][DEBUG ] connected to host: master-arm-1
   [master-arm-1][INFO  ] Running command: sudo ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.master-arm-1.asok mon_status
   [ceph_deploy.mon][INFO  ] mon.master-arm-1 monitor has reached quorum!
   [ceph_deploy.mon][INFO  ] all initial monitors are running and have formed quorum
   [ceph_deploy.mon][INFO  ] Running gatherkeys...
   [ceph_deploy.gatherkeys][INFO  ] Storing keys in temp directory /tmp/tmp603jq7ge
   [master-arm-1][DEBUG ] connection detected need for sudo
   [master-arm-1][DEBUG ] connected to host: master-arm-1
   [master-arm-1][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --admin-daemon=/var/run/ceph/ceph-mon.master-arm-1.asok mon_status
   [master-arm-1][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-master-arm-1/keyring auth get client.admin
   [master-arm-1][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-master-arm-1/keyring auth get client.bootstrap-mds
   [master-arm-1][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-master-arm-1/keyring auth get client.bootstrap-mgr
   [master-arm-1][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-master-arm-1/keyring auth get client.bootstrap-osd
   [master-arm-1][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-master-arm-1/keyring auth get client.bootstrap-rgw
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mgr.keyring
   [ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
   [ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
   [ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmp603jq7ge
   ```

5. 分发4中的密钥，方便`ceph CLI`的使用
   
   ```
   ceph-deploy admin worker-amd64-gpuceph-node1 worker-amd64-gpuceph-node2 worker-amd64-gpuceph-node3
   ```

6. 部署管理daemon，对于luminous版本之上的ceph需要
   
   ```
   ceph-deploy mgr create master-arm-1
   ```

7. 添加3个OSD，~~OSD部署磁盘必须为裸盘~~OSD可部署在多种设备上，详见[链接](https://docs.ceph.com/docs/master/ceph-volume/lvm/prepare/#ceph-volume-lvm-prepare-filestore)
   
   ```
   ceph-deploy osd create --data ubuntu-vg/ceph-volume-1 worker-amd64-gpuceph-node1
   ceph-deploy osd create --data ubuntu-vg/ceph-volume-2 worker-amd64-gpuceph-node1
   ceph-deploy osd create --data ubuntu-vg/ceph-volume-3 worker-amd64-gpuceph-node1
   ...
   ceph-deploy osd create --data ubuntu-vg/ceph-volume-6 worker-amd64-gpuceph-node3
   ```

8. 在worker-amd64-1上查看lvm情况，可看到对于ceph而言，其osd后端仍然使用的是lvm支撑。**注：此处为使用裸盘部署osd的情况，使用lvm的话可观察/var/lib/ceph下的内容了解ceph对lvm的使用**
   
   ```
   pvs
     PV             VG                                        Fmt  Attr PSize   PFree
     /dev/nvme0n1p3 ceph-f453363b-d90a-40aa-a58c-1a5834409b7f lvm2 a--  <40.00g    0
     /dev/nvme0n1p4 ceph-0c217efd-00ca-4964-a7b0-8188358e8483 lvm2 a--  <40.00g    0
     /dev/nvme0n1p5 ceph-401df6a7-36f3-496d-a314-211a2f3d2bef lvm2 a--  <40.00g    0
   ```

#### ADD more monitors


1. 添加新的monitors
   
   ```
   ceph-deploy mon add worker-amd64-gpuceph-node2 
   ceph-deploy mon add worker-amd64-gpuceph-node3
   ```

2. 查看此时选举状态
   
   ```
   # ssh worker-amd64-gpuceph-node1 sudo ceph quorum_status --format json-pretty
   ```

   ```json
   {
       "election_epoch": 12,
       "quorum": [
           0,
           1,
           2
       ],
       "quorum_names": [
           "worker-amd64-gpuceph-node1",
           "worker-amd64-gpuceph-node2",
           "worker-amd64-gpuceph-node3"
       ],
       "quorum_leader_name": "worker-amd64-gpuceph-node1",
       "quorum_age": 28,
       "features": {
           "quorum_con": "4540138292836696063",
           "quorum_mon": [
               "kraken",
               "luminous",
               "mimic",
               "osdmap-prune",
               "nautilus",
               "octopus"
           ]
       },
       "monmap": {
           "epoch": 3,
           "fsid": "806762f5-9823-4f93-9afa-bd627436e0c8",
           "modified": "2020-08-21T12:05:20.726616Z",
           "created": "2020-08-21T11:48:39.391492Z",
           "min_mon_release": 15,
           "min_mon_release_name": "octopus",
           "features": {
               "persistent": [
                   "kraken",
                   "luminous",
                   "mimic",
                   "osdmap-prune",
                   "nautilus",
                   "octopus"
               ],
               "optional": []
           },
           "mons": [
               {
                   "rank": 0,
                   "name": "worker-amd64-gpuceph-node1",
                   "public_addrs": {
                       "addrvec": [
                           {
                               "type": "v2",
                               "addr": "10.10.197.200:3300",
                               "nonce": 0
                           },
                           {
                               "type": "v1",
                               "addr": "10.10.197.200:6789",
                               "nonce": 0
                           }
                       ]
                   },
                   "addr": "10.10.197.200:6789/0",
                   "public_addr": "10.10.197.200:6789/0",
                   "priority": 0,
                   "weight": 0
               },
               {
                   "rank": 1,
                   "name": "worker-amd64-gpuceph-node2",
                   "public_addrs": {
                       "addrvec": [
                           {
                               "type": "v2",
                               "addr": "10.10.197.201:3300",
                               "nonce": 0
                           },
                           {
                               "type": "v1",
                               "addr": "10.10.197.201:6789",
                               "nonce": 0
                           }
                       ]
                   },
                   "addr": "10.10.197.201:6789/0",
                   "public_addr": "10.10.197.201:6789/0",
                   "priority": 0,
                   "weight": 0
               },
               {
                   "rank": 2,
                   "name": "worker-amd64-gpuceph-node3",
                   "public_addrs": {
                       "addrvec": [
                           {
                               "type": "v2",
                               "addr": "10.10.197.202:3300",
                               "nonce": 0
                           },
                           {
                               "type": "v1",
                               "addr": "10.10.197.202:6789",
                               "nonce": 0
                           }
                       ]
                   },
                   "addr": "10.10.197.202:6789/0",
                   "public_addr": "10.10.197.202:6789/0",
                   "priority": 0,
                   "weight": 0
               }
           ]
       }
   }
   ```

3. 添加manager daemon，现实新增的两个manager daemon处于standby状态
   
   ```
   $ ceph-deploy mgr create worker-amd64-gpuceph-node2 worker-amd64-gpuceph-node3
   $ worker-amd64-gpuceph-node3
     cluster:
       id:     806762f5-9823-4f93-9afa-bd627436e0c8
       health: HEALTH_OK

     services:
       mon: 3 daemons, quorum worker-amd64-gpuceph-node1,worker-amd64-gpuceph-node2,worker-amd64-gpuceph-node3 (age 4m)
       mgr: worker-amd64-gpuceph-node1(active, since 18m), standbys: worker-amd64-gpuceph-node2, worker-amd64-gpuceph-node3
       osd: 18 osds: 18 up (since 11m), 18 in (since 11m)

     data:
       pools:   1 pools, 1 pgs
       objects: 0 objects, 0 B
       usage:   18 GiB used, 1.7 TiB / 1.8 TiB avail
       pgs:     1 active+clean
   ```

#### ADD RGW INSTANCE

为使用Ceph的`Ceph Object Gateway`组件，需要在ceph上部署RGW实例

1. 使用如下指令
   
   ```
   # ceph-deploy rgw create worker-amd64-gpuceph-node1
   ```

2. 验证网关服务是否部署成功
   
   ```
   # curl http://worker-amd64-gpuceph-node1:7480
   ```

   ```xml
   <?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>
   ```

默认状态下`Ceph Object Gateway`使用7480端口，但可通过修改运行节点的ceph.conf的`[client]`属性进行配置

```
[client]
rgw frontends = civetweb port=80
```

### TEST

***



1. 创建新的ceph存储池
   
   ```
   # sudo ceph osd pool create testpool
   # sudo ceph df
   --- RAW STORAGE ---
   CLASS  SIZE     AVAIL    USED     RAW USED  %RAW USED
   hdd    1.8 TiB  1.7 TiB  521 MiB    19 GiB       1.03
   TOTAL  1.8 TiB  1.7 TiB  521 MiB    19 GiB       1.03

   --- POOLS ---
   POOL                   ID  STORED   OBJECTS  USED     %USED  MAX AVAIL
   device_health_metrics   1      0 B        0      0 B      0    564 GiB
   .rgw.root               2  1.2 KiB        4  768 KiB      0    564 GiB
   default.rgw.log         3  3.4 KiB      175    6 MiB      0    564 GiB
   default.rgw.control     4      0 B        8      0 B      0    564 GiB
   default.rgw.meta        5      0 B        0      0 B      0    564 GiB
   testpool                6      6 B        1  192 KiB      0    564 GiB
   ```

2. 创建新的文件，并上传至1创建的存储池中
   
   ```
   # echo "hello" >  hello.txt
   # sudo rados put test-object-1 hello.txt --pool=testpool
   # sudo rados -p testpool ls
   test-object-1
   ```

3. 查看对象分布
   
   ```
   # sudo ceph osd map testpool test-object-1
   osdmap e227 pool 'testpool' (6) object 'test-object-1' -> pg 6.74dc35e2 (6.2) -> up ([7,4,15], p7) acting ([7,4,15], p7)
   ```

4. 删除test-object-1对象
   
   ```
   # sudo rados rm test-object-1 --pool=testpool
   # sudo rados -p testpool ls
   ```

5. 删除testpool存储池
   
   ```
   # sudo ceph osd pool rm testpool
   Error EPERM: WARNING: this will *PERMANENTLY DESTROY* all data stored in pool testpool.  If you are *ABSOLUTELY CERTAIN* that is what you want, pass the pool name *twice*, followed by --yes-i-really-really-mean-it.
   ```
