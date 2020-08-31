- [Ceph As Volume Backend](#ceph-as-volume-backend)
  - [Kubernetes/ Ceph Technology Stack](#kubernetes-ceph-technology-stack)
  - [Procedure](#procedure)
    - [Stage 1 CREATE A POOL](#stage-1-create-a-pool)
    - [Stage 2 CONFIGURE CEPH_CSI](#stage-2-configure-ceph_csi)
      - [SETUP CEPH CLIENT AUTHENTICATION](#setup-ceph-client-authentication)
      - [GENERATE CEPH-CSI CONFIGMAP](#generate-ceph-csi-configmap)

# Ceph As Volume Backend

## Kubernetes/ Ceph Technology Stack

![Alt Text](pictures/Kubernetes%20use%20Ceph%20Technology%20Stack.png)

Kubernetes 1.13版本后通过ceph-csi使用`Ceph Block Device Images`作为存储后端，`ceph-csi`动态地提供`RBD images`作为Kubernetes volumes的存储后端，**并将`RBD images`作为运行着需要RBD-backed volumePods的Kubernetes的worker nodes的block devices**（可选，在block service上挂载文件系统）。

通过使用Ceph作为Kubernetes的存储后端，在一定程度上可利用Ceph的分布式存储特性，提高读写速率。

***

## Procedure

### Stage 1 CREATE A POOL

1. 使用ceph命令创建osd存储池

    ```
    $ sudo ceph osd pool create kubernetes
    pool 'kubernetes' created
    $ sudo ceph df
      --- RAW STORAGE ---
      CLASS  SIZE     AVAIL    USED     RAW USED  %RAW USED
      hdd    1.8 TiB  1.7 TiB  651 MiB    19 GiB       1.04
      TOTAL  1.8 TiB  1.7 TiB  651 MiB    19 GiB       1.04

      --- POOLS ---
      POOL                   ID  STORED   OBJECTS  USED     %USED  MAX AVAIL
      device_health_metrics   1      0 B        3      0 B      0    564 GiB
      .rgw.root               2  1.2 KiB        4  768 KiB      0    564 GiB
      default.rgw.log         3  3.4 KiB      207    6 MiB      0    564 GiB
      default.rgw.control     4      0 B        8      0 B      0    564 GiB
      default.rgw.meta        5      0 B        0      0 B      0    564 GiB
      testpool                6      0 B        0      0 B      0    564 GiB
      kubernetes              7      0 B        0      0 B      0    564 GiB
    ```

2. 使用rbd命令初始化rbd存储池
   
   ```
   $ sudo rbd pool init kubernetes
   ```

### Stage 2 CONFIGURE CEPH_CSI

####  SETUP CEPH CLIENT AUTHENTICATION

1. 创建一个新的Ceph client用户，用于Kubernetes和Ceph-csi，运行下述命令
   
   ```
   $ sudo ceph get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
   [client.kubernetes]
           key = AQC/gURf+/B1OBAABFutKMKlcXls8wVnRoIFUA==
   ```

#### GENERATE CEPH-CSI CONFIGMAP

`ceph-csi`需要一个`ConfigMap`对象，该对象将被存储在Kubernetes中，用于定义Ceph monitor的地址

1. 获取ceph cluster的fsid以及各个monitor的地址，`ceph-csi`仅支持[legacy vi protocol](https://docs.ceph.com/docs/master/rados/configuration/msgr2/#address-formats)
   
   ```
   $ sudo ceph mon dump
   dumped monmap epoch 3
   epoch 3
   fsid 806762f5-9823-4f93-9afa-bd627436e0c8
   last_changed 2020-08-21T20:05:20.726616+0800
   created 2020-08-21T19:48:39.391492+0800
   min_mon_release 15 (octopus)
   0: [v2:10.10.197.200:3300/0,v1:10.10.197.200:6789/0] mon.worker-amd64-gpuceph-node1
   1: [v2:10.10.197.201:3300/0,v1:10.10.197.201:6789/0] mon.worker-amd64-gpuceph-node2
   2: [v2:10.10.197.202:3300/0,v1:10.10.197.202:6789/0] mon.worker-amd64-gpuceph-node3
   ```

2. 编辑`csi-config-map.yaml`，如下所示
   
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   data:
        config.json: |-
            [
                {
                    "clusterID": "806762f5-9823-4f93-9afa-bd627436e0c8",
                    "monitors": [
                        "10.10.197.200:6789",
                        "10.10.197.201:6789",
                        "10.10.197.202:6789"
                    ]
                }
            ]
   metadata:
        name: ceph-csi-config
   ```

3. 使用kubectl命令创建ConfigMap对象
   
   ```
   
   ```
