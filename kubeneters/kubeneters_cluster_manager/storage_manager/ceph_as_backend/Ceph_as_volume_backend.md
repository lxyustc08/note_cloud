- [Ceph As Volume Backend](#ceph-as-volume-backend)
  - [Kubernetes/ Ceph Technology Stack](#kubernetes-ceph-technology-stack)
  - [Procedure](#procedure)
    - [Stage 1 CREATE A POOL](#stage-1-create-a-pool)
    - [Stage 2 CONFIGURE CEPH_CSI](#stage-2-configure-ceph_csi)
      - [SETUP CEPH CLIENT AUTHENTICATION](#setup-ceph-client-authentication)
      - [GENERATE CEPH-CSI CONFIGMAP](#generate-ceph-csi-configmap)
      - [GENERATE CEPH-CSI CEPHX SECRET](#generate-ceph-csi-cephx-secret)
      - [CINFUGURE CEPH-CSI PLUGINS](#cinfugure-ceph-csi-plugins)
    - [Stage 3 Testing Ceph csi with Pod Using as file](#stage-3-testing-ceph-csi-with-pod-using-as-file)
    - [Stage 4 Testing Ceph csi with Pod as raw block](#stage-4-testing-ceph-csi-with-pod-as-raw-block)

# Ceph As Volume Backend

> 注：本处采用的provisioner为ceph开发的ceph csi而非kubernetes内置的rbd provisioner，[可实现raw block volume support](../../../kubernetes_concept/kubernetes_storage/volume.md)

## Kubernetes/ Ceph Technology Stack

![Alt Text](../../pictures/Kubernetes%20use%20Ceph%20Technology%20Stack.png)

Kubernetes 1.13版本后通过ceph-csi使用`Ceph Block Device Images`作为存储后端，`ceph-csi`动态地提供`RBD images`作为Kubernetes volumes的存储后端，**并将`RBD images`作为运行着需要RBD-backed volumePods的Kubernetes的worker nodes的block devices**（可选，在block service上挂载文件系统）。

通过使用Ceph作为Kubernetes的存储后端，在一定程度上可利用Ceph的分布式存储特性，提高读写速率。

***

## Procedure

本过程基于如下软件配置进行：

|序号|软件名称|版本|
|:---:|:---:|:---:|
|1|ceph cluster|15.2.9|
|2|kubernetes|1.20.4|
|3|ceph csi|v3.2|

本次过程使用到的镜像如下：

|序号|镜像名称|版本|类型|
|:---:|:---:|:---:|:---:|
|1|csi-attacher|v3.0.2|sidecar镜像|
|2|csi-provisioner|v2.0.4|sidecar镜像|
|3|csi-snapshotter|v3.0.2|sidecar镜像|
|4|csi-resizer|v1.0.1|sidecar镜像|
|5|csi-node-driver-registrar|v2.0.1|sidecar镜像|
|6|cephcsi|v3.2-canary|ceph csi镜像|

> 本过程使用ceph的rbd作为kubernetes的后端存储

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
   $ sudo ceph get-or-create client.kubernetes.block mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
   [client.kubernetes]
           key = AQCG82pf3Q0xARAAcbwPn7B0xrr25FKLtZ03Hw==
   ```

#### GENERATE CEPH-CSI CONFIGMAP

`ceph-csi`需要一个`ConfigMap`对象，该对象将被存储在Kubernetes中，用于定义Ceph monitor的地址

1. 获取ceph cluster的fsid以及各个monitor的地址，`ceph-csi`仅支持[legacy vi protocol](https://docs.ceph.com/docs/master/rados/configuration/msgr2/#address-formats)
   
   ```
   $ ceph mon dump
   dumped monmap epoch 3
   epoch 3
   fsid bc1f0b56-f252-11ea-8930-a9753a839177
   last_changed 2020-09-09T04:19:38.978756+0000
   created 2020-09-09T04:13:02.686969+0000
   min_mon_release 15 (octopus)
   0: [v2:10.10.197.200:3300/0,v1:10.10.197.200:6789/0] mon.worker-amd64-gpuceph-node1
   1: [v2:10.10.197.201:3300/0,v1:10.10.197.201:6789/0] mon.worker-amd64-gpuceph-node2
   2: [v2:10.10.197.202:3300/0,v1:10.10.197.202:6789/0] mon.worker-amd64-gpuceph-node3
   ```

2. 编辑`csi-config-map.yaml`，如下所示
   
   ```yaml
   ---
   apiVersion: v1
   kind: ConfigMap
   data:
        config.json: |-
            [
                {
                    "clusterID": "bc1f0b56-f252-11ea-8930-a9753a839177",
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

3. 使用kubectl命令创建[ConfigMap对象](../kubernetes_concept/kubernetes_configuration/kubernetes_config_maps.md)，此处需要指定`namespace`，后续的plugin部署时的`namespace`需与之保持一致，且创建的相关角色也需放置在该`namespace`中
   
   ```
   $ kubectl apply -f csi-config-map.yaml -n default
   configmap/ceph-csi-config created
   $ kubectl get configmaps -n default
   NAME              DATA   AGE
   ceph-csi-config   1      17h
   ```

4. ceph-csi的master分支部署时需要一个额外的ConfigMap对象，该对象用以指定数据加密相关细节，主要针对[KMS](https://en.wikipedia.org/wiki/Key_management#Key_management_system)（Key Management System）密钥管理工具vault而言，内容如下：
   
   ```yaml
   ---
   apiVersion: v1
   kind: ConfigMap
   data:
     config.json: |-
       {
         "vault-test": {
           "encryptionKMSType": "vault",
           "vaultAddress": "http://vault.default.svc.cluster.local:8200",
           "vaultAuthPath": "/v1/auth/kubernetes/login",
           "vaultRole": "csi-kubernetes",
           "vaultPassphraseRoot": "/v1/secret",
           "vaultPassphrasePath": "ceph-csi/",
           "vaultCAVerify": "false"
         }
       }
   metadata:
     name: ceph-csi-encryption-kms-config
   ```

5. 使用命令创建ceph-csi-encryption-kms-config ConfigMap对象
   
   ```
   $ kubectl apply -f ceph-csi-encryption-kms-config.yaml -n default 
   configmap/ceph-csi-encryption-kms-config created
   $ kubectl get configmaps
   NAME                             DATA   AGE
   ceph-csi-config                  1      7d16h
   ceph-csi-encryption-kms-config   1      87s
   ```

#### GENERATE CEPH-CSI CEPHX SECRET

ceph-csi需要使用cephx credentials以用于与Ceph cluster进行通信

1. 使用生成的[Ceph client authentication](Ceph_as_volume_backend.md#setup-ceph-client-authentication)以及用户名创建`csi-rbd-secret.yaml`文件
   
   ```yaml
   ---
   apiVersion: v1
   kind: Secret
   metadata: 
        name: csi-rbd-secret
        namespace: default
   stringData:
        userID: kubernetes.block
        userKey: AQCG82pf3Q0xARAAcbwPn7B0xrr25FKLtZ03Hw==
   ```

2. 使用kubectl命令创建secret对象
   
   ```
   $ kubectl apply -f csi-rbd-secret.yaml -n default
   secret/csi-rbd-secret created
   $ kubectl get secrets
   NAME                  TYPE                                  DATA   AGE
   csi-rbd-secret        Opaque                                2      67s
   default-token-zhxhg   kubernetes.io/service-account-token   3      102d   
   ```

#### CINFUGURE CEPH-CSI PLUGINS

1. 创建必要的`ServiceAccount`以及必要的`RBAC ClusterRole/ClusterRoleBinding` Kubernetes对象
   
   ```
   $ wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
   $ wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
   $ kubectl apply -f csi-nodeplugin-rbac.yaml
   serviceaccount/rbd-csi-nodeplugin created
   clusterrole.rbac.authorization.k8s.io/rbd-csi-nodeplugin created
   clusterrolebinding.rbac.authorization.k8s.io/rbd-csi-nodeplugin created
   $ kubectl apply -f csi-provisioner-rbac.yaml
   serviceaccount/rbd-csi-provisioner created
   clusterrole.rbac.authorization.k8s.io/rbd-external-provisioner-runner created
   clusterrolebinding.rbac.authorization.k8s.io/rbd-csi-provisioner-role created
   role.rbac.authorization.k8s.io/rbd-external-provisioner-cfg created
   rolebinding.rbac.authorization.k8s.io/rbd-csi-provisioner-role-cfg created   
   ```

2. 获取部署的yaml文件
   
   ```
   $ wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
   $ wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
   ```

   上述yaml文件中的image均修改为本地镜像地址

3. 部署插件（-n default参数可以不进行指定，在插件配置yaml文件中已经指定了namespace参数）
   
   ```
   $ kubectl apply -f csi-rbdplugin-provisioner.yaml -n default
   service/csi-rbdplugin-provisioner created
   deployment.apps/csi-rbdplugin-provisioner created
   $ kubectl apply -f csi-rbdplugin.yaml -n default
   daemonset.apps/csi-rbdplugin created
   service/csi-metrics-rbdplugin created
   ```

4. 查看ceph-csi部署pods运行状态
   
   ```
   $ kubectl get pods
   csi-rbd-demo-pod                             1/1     Running     0          27m
   csi-rbdplugin-2wkl2                          3/3     Running     0          37m
   csi-rbdplugin-4q2ck                          3/3     Running     0          37m
   csi-rbdplugin-8qhrt                          3/3     Running     0          37m
   csi-rbdplugin-lsngn                          3/3     Running     0          37m
   csi-rbdplugin-provisioner-5b7c6664fd-5q2d5   7/7     Running     0          37m
   csi-rbdplugin-provisioner-5b7c6664fd-glb6z   7/7     Running     0          37m
   csi-rbdplugin-provisioner-5b7c6664fd-r66s9   7/7     Running     0          37m
   ```

   上述信息表明ceph-csi插件相关pods已经成功运行，但整体ceph-csi服务是否已经完全运行正常还需通过pvc是否可正常工作来进行验证。

### Stage 3 Testing Ceph csi with Pod Using as file

本部分通过创建使用pvc作为文件夹存储来测试ceph csi是否正常工作

1. 创建storageClass，描述文件storageclass.yaml如下：
   
   ```yaml
   ---
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
      name: csi-rbd-sc
   provisioner: rbd.csi.ceph.com
   # If topology based provisioning is desired, delayed provisioning of
   # PV is required and is enabled using the following attribute
   # For further information read TODO<doc>
   # volumeBindingMode: WaitForFirstConsumer
   parameters:
      clusterID: bc1f0b56-f252-11ea-8930-a9753a839177

      pool: kubernetes

      imageFeatures: layering

      csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
      csi.storage.k8s.io/provisioner-secret-namespace: default
      csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
      csi.storage.k8s.io/controller-expand-secret-namespace: default
      csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
      csi.storage.k8s.io/node-stage-secret-namespace: default

      csi.storage.k8s.io/fstype: ext4

   reclaimPolicy: Delete
   allowVolumeExpansion: true
   mountOptions:
      - discard
   ```

2. 查看storageClass
   
   ```
   kubectl get sc
   NAME         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
   csi-rbd-sc   rbd.csi.ceph.com   Delete          Immediate           true                   50m
   ```

3. 创建pvc，其描述文件pvc.yaml如下
   
   ```yaml
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: rbd-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
     storageClassName: csi-rbd-sc
   ```

4. 查看pvc
   
   ```
   kubectl get pvc
   NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   rbd-pvc   Bound    pvc-10765d6a-eefb-4108-a3a0-4a0574f4610c   1Gi        RWO            csi-rbd-sc     49m
   ```

   上述状态表明pvc创建成功

5. 创建pod，其描述文件pod.yaml内容如下
   
   ```yaml
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: csi-rbd-demo-pod
   spec:
     containers:
       - name: web-server
         image: docker.io/library/nginx:latest
         volumeMounts:
           - name: mypvc
             mountPath: /var/lib/www/html
     volumes:
       - name: mypvc
         persistentVolumeClaim:
           claimName: rbd-pvc
           readOnly: false   
   ```

6. 查看pod状态
   
   ```
   kubectl get pods
   NAME                                         READY   STATUS      RESTARTS   AGE
   csi-rbd-demo-pod                             1/1     Running     0          49m
   ```

   上述状态表明，ceph csi可正常运行

### Stage 4 Testing Ceph csi with Pod as raw block

本部分通过创建使用pvc作为raw block的pod测试ceph csi是否工作正常，本部分使用同Stage3的storageClass

1. 创建pvc，其描述文件如下
   
   ```yaml
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: raw-block-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     volumeMode: Block
     resources:
       requests:
         storage: 1Gi
     storageClassName: csi-rbd-sc
   ```

2. 查看pvc
   
   ```
   kubectl get pvc
   NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   raw-block-pvc   Bound    pvc-2f83a013-7345-47e3-9cc2-5105c58e2316   1Gi        RWO            csi-rbd-sc     4s
   ```

   上述输出说明pvc创建成功

3. 创建使用上述pvc的pod，其描述文件如下：
   
   ```yaml
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-with-raw-block-volume
   spec:
     containers:
       - name: centos
         image: registry.centos.org/centos:latest
         command: ["/bin/sleep", "infinity"]
         volumeDevices:
           - name: data
             devicePath: /dev/xvda
     volumes:
       - name: data
         persistentVolumeClaim:
           claimName: raw-block-pvc
   ```

4. 查看pod状态
   
   ```
   kubectl get pod
   ......
   pod-with-raw-block-volume                    1/1     Running     0          8m19s
   ```

5. 查看pod中raw block的状态
   
   ```
   kubectl exec pod-with-raw-block-volume -- fdisk -l /dev/xvda
   Disk /dev/xvda: 1 GiB, 1073741824 bytes, 2097152 sectors
   Units: sectors of 1 * 512 = 512 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 65536 bytes / 65536 bytes
   ```

   上述输出表明，raw block已经成功挂载至pod中

