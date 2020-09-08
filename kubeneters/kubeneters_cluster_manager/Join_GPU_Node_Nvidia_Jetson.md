- [Join GPU Node Jetson Nano To Kubernetes Cluster](#join-gpu-node-jetson-nano-to-kubernetes-cluster)
  - [Overview](#overview)
  - [Environment](#environment)
  - [Procedures](#procedures)
    - [Step1 Setup container runtime](#step1-setup-container-runtime)
    - [Step2 Build nvidia device plugin image for arm64](#step2-build-nvidia-device-plugin-image-for-arm64)
    - [Step3 nvidia k8s device plugin deployment](#step3-nvidia-k8s-device-plugin-deployment)
  - [Testing](#testing)
    - [基本资源调度验证](#基本资源调度验证)
    - [资源竞争验证](#资源竞争验证)
  - [容器运行时要求](#容器运行时要求)

# Join GPU Node Jetson Nano To Kubernetes Cluster

## Overview

采用Kubernetes的[device plugin](../kubernetes_concept/kubernetes_plugin/device_plugin.md)机制，将GPU资源纳入kubernetes管理。

## Environment

|IP|Role|ARCH|hostname|
|:---:|:---:|:---:|:---:|
|10.10.197.101| kubernetes controller plane| ARM 64 X 3|master-arm-1</br>master-arm-2</br>master-arm-3|
|10.10.197.94|gpu node|ARM 64 with nvidia GPU (Jetson nano)|worker-gpu-node1|
|10.10.197.93|gpu node|ARM 64 with nvidia GPU (Jetson nano)|worker-gpu-node2|

## Procedures

### Step1 Setup container runtime

参考[此处](../../AI/AI_environment/nvidia_jetson_nano_config.md)，对nvidia jetson nano进行配置，设置container runtime


### Step2 Build nvidia device plugin image for arm64

nvidia官方仅提供AMD64与PPC64LE官方容器镜像，未提供ARM64镜像，且如果直接手动部署，也会报错缺少NVML，参考此篇[文档](https://blogs.windriver.com/wind_river_blog/2020/06/nvidia-k8s-device-plugin-for-wind-river-linux/)，对官方进行补丁支持，使之支持ARM64架构。

1. 下载nvidia k8s plugin源码
   
   ```
   # git clone https://github.com/NVIDIA/k8s-device-plugin.git
   # cd k8s-device-plugin
   ```

2. 切换至1.0.0-beta6 tag
   
   ```
   # git checkout -b 1.0.0-beta6 1.0.0-beta6
   ```

3. 下载补丁
   
   ```
   # wget https://labs.windriver.com/downloads/0001-arm64-add-support-for-arm64-architectures.patch
   # wget https://labs.windriver.com/downloads/0002-nvidia-Add-support-for-tegra-boards.patch
   # wget https://labs.windriver.com/downloads/0003-main-Add-support-for-tegra-boards.patch
   ```

4. 打补丁
   
   ```
   # git am 000*.patch
   ```

5. 修改docker/arm64/Dockerfile.ubuntu16.04(打补丁后生成的新文件)，修改内容包括，更改golang程序下载镜像，添加GOPROXY环境变量
   
   ```Dockerfile
   FROM --platform=arm64 ubuntu:16.04 as build

   RUN apt-get update && apt-get install -y --no-install-recommends \
           g++ \
           ca-certificates \
           wget && \
       rm -rf /var/lib/apt/lists/*

   ENV GOLANG_VERSION 1.10.3
   RUN wget -nv -O - http://mirrors.ustc.edu.cn/golang/go${GOLANG_VERSION}.linux-arm64.tar.gz \
       | tar -C /usr/local -xz
   ENV GOPATH /go
   ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
   ENV GOPROXY "https://goproxy.cn,direct"

   WORKDIR /go/src/nvidia-device-plugin
   COPY . .

   RUN export CGO_LDFLAGS_ALLOW='-Wl,--unresolved-symbols=ignore-in-object-files' && \
       go install -ldflags="-s -w" -v nvidia-device-plugin


   FROM --platform=arm64 debian:stretch-slim

   ENV NVIDIA_VISIBLE_DEVICES=all
   ENV NVIDIA_DRIVER_CAPABILITIES=utility

   COPY --from=build /go/bin/nvidia-device-plugin /usr/bin/nvidia-device-plugin

   CMD ["nvidia-device-plugin"]
   ```

6. 构建容器镜像
   
   ```
   # docker buil --tag nvidia/k8s-device-plugin-arm64:1.0.0-beta6 --file docker/arm64/Dockerfile.ubuntu16.04 .
   ```

7. 将容器镜像上传至自建仓库中
   
   ```
   # docker tag nvidia/k8s-device-plugin-arm64:1.0.0-beta6 lxyustc.registrydomain.com:5000/k8s-device-plugin-arm64:1.0.0-beta6
   # docker push lxyustc.registrydomain.com:5000/k8s-device-plugin-arm64:1.0.0-beta6
   ```

8. 在镜像仓库机器10.10.197.96上建立镜像的manifest，并上传至镜像仓库中, (`lxyustc.registrydomain.com:5000/k8s-device-plugin-amd64:1.0.0-beta6`镜像为nvidia官方amd64镜像)
   
   ```
   # docker manifest create lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6 lxyustc.registrydomain.com:5000/k8s-device-plugin-arm64:1.0.0-beta6 lxyustc.registrydomain.com:5000/k8s-device-plugin-amd64:1.0.0-beta6
   # docker manifest annotate lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6 lxyustc.registrydomain.com:5000/k8s-device-plugin-arm64:1.0.0-beta6 --os linux --arch arm64
   # docker manifest annotate lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6 lxyustc.registrydomain.com:5000/k8s-device-plugin-amd64:1.0.0-beta6 --os linux --arch amd64
   # docker manifest push lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6
   ```

9. 查看8中镜像的manifest
    
   ```
   # docker manifest inspect lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6
   {
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 740,
         "digest": "sha256:00c700122ebc5533e87bf2df193f457d2c2ee37a4a97999466a9a388617cb16b",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 740,
         "digest": "sha256:caa30f3954ddd2d90aeb69bb6035598712fa56bf3fb3d2e64f16d5f056e668c6",
         "platform": {
            "architecture": "arm64",
            "os": "linux"
         }
      }
    ]
   }
   ```

### Step3 nvidia k8s device plugin deployment

采用DaemonSet的方式进行部署

1. 获取DaemonSet描述yaml文件
   
   ```
   wget https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.6.0/nvidia-device-plugin.yml
   ```

2. 修改获取的nvidia-device-plugin，主要修改镜像值，指向前一步骤中生成的本地仓库镜像
   
   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: nvidia-device-plugin-daemonset
     namespace: kube-system
   spec:
     selector:
       matchLabels:
         name: nvidia-device-plugin-ds
     updateStrategy:
       type: RollingUpdate
     template:
       metadata:
         # This annotation is deprecated. Kept here for backward compatibility
         # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
         annotations:
           scheduler.alpha.kubernetes.io/critical-pod: ""
         labels:
           name: nvidia-device-plugin-ds
       spec:
         tolerations:
         # This toleration is deprecated. Kept here for backward compatibility
         # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
         - key: CriticalAddonsOnly
           operator: Exists
         - key: nvidia.com/gpu
           operator: Exists
           effect: NoSchedule
         # Mark this pod as a critical add-on; when enabled, the critical add-on
         # scheduler reserves resources for critical add-on pods so that they can
         # be rescheduled after a failure.
         # See https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
         priorityClassName: "system-node-critical"
         containers:
         - image: lxyustc.registrydomain.com:5000/k8s-device-plugin:1.0.0-beta6
           name: nvidia-device-plugin-ctr
           securityContext:
             allowPrivilegeEscalation: false
             capabilities:
               drop: ["ALL"]
           volumeMounts:
             - name: device-plugin
               mountPath: /var/lib/kubelet/device-plugins
         volumes:
           - name: device-plugin
             hostPath:
               path: /var/lib/kubelet/device-plugins
   ```

3. 使用命令部署DaemonSet
   
   ```
   # kubectl apply -f nvidia-device-plugin.yml
   ```

4. 验证worker-gpu-node1，在Capacity一栏中出现名称为 `nvidia.com/gpu`， 数量为1的GPU资源
   
   ```
   # kubectl describe nodes worker-gpu-node1
   Name:               worker-gpu-node1
   Roles:              <none>
   Labels:             beta.kubernetes.io/arch=arm64
                       beta.kubernetes.io/os=linux
                       kubernetes.io/arch=arm64
                       kubernetes.io/hostname=worker-gpu-node1
                       kubernetes.io/os=linux
   Annotations:        flannel.alpha.coreos.com/backend-data: {"VtepMAC":"2e:d9:20:31:5e:57"}
                       flannel.alpha.coreos.com/backend-type: vxlan
                       flannel.alpha.coreos.com/kube-subnet-manager: true
                       flannel.alpha.coreos.com/public-ip: 10.10.197.94
                       kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                       node.alpha.kubernetes.io/ttl: 0
                       volumes.kubernetes.io/controller-managed-attach-detach: true
   CreationTimestamp:  Thu, 30 Jul 2020 18:37:48 +0800
   Taints:             <none>
   Unschedulable:      false
   Lease:
     HolderIdentity:  worker-gpu-node1
     AcquireTime:     <unset>
     RenewTime:       Tue, 04 Aug 2020 15:01:37 +0800
   Conditions:
     Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
     ----                 ------  -----------------                 ------------------                ------                       -------
     NetworkUnavailable   False   Fri, 31 Jul 2020 22:11:28 +0800   Fri, 31 Jul 2020 22:11:28 +0800   FlannelIsUp                  Flannel is running on this node
     MemoryPressure       False   Tue, 04 Aug 2020 14:58:44 +0800   Thu, 30 Jul 2020 18:37:48 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
     DiskPressure         False   Tue, 04 Aug 2020 14:58:44 +0800   Thu, 30 Jul 2020 18:37:48 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
     PIDPressure          False   Tue, 04 Aug 2020 14:58:44 +0800   Thu, 30 Jul 2020 18:37:48 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
     Ready                True    Tue, 04 Aug 2020 14:58:44 +0800   Fri, 31 Jul 2020 22:11:15 +0800   KubeletReady                 kubelet is posting ready status
   Addresses:
     InternalIP:  10.10.197.94
     Hostname:    worker-gpu-node1
   Capacity:
     cpu:                4
     ephemeral-storage:  30625592Ki
     hugepages-2Mi:      0
     memory:             4058068Ki
     nvidia.com/gpu:     1
     pods:               110
   Allocatable:
     cpu:                4
     ephemeral-storage:  28224545541
     hugepages-2Mi:      0
     memory:             3955668Ki
     nvidia.com/gpu:     1
     pods:               110
   System Info:
     Machine ID:                 a3d9197b765643568af09eb2bd3e5ce7
     System UUID:                a3d9197b765643568af09eb2bd3e5ce7
     Boot ID:                    2e9c127c-3ff0-4d40-9ad2-186a1d3785a8
     Kernel Version:             4.9.140-tegra
     OS Image:                   Ubuntu 18.04.4 LTS
     Operating System:           linux
     Architecture:               arm64
     Container Runtime Version:  docker://19.3.6
     Kubelet Version:            v1.18.6
     Kube-Proxy Version:         v1.18.6
   PodCIDR:                      10.88.5.0/24
   PodCIDRs:                     10.88.5.0/24
   Non-terminated Pods:          (3 in total)
     Namespace                   Name                                    CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
     ---------                   ----                                    ------------  ----------  ---------------  -------------  ---
     kube-system                 kube-flannel-ds-arm64-fmpvm             100m (2%)     100m (2%)   50Mi (1%)        50Mi (1%)      4d20h
     kube-system                 kube-proxy-sh6bl                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         4d20h
     kube-system                 nvidia-device-plugin-daemonset-d9999    0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d14h
   Allocated resources:
     (Total limits may be over 100 percent, i.e., overcommitted.)
     Resource           Requests   Limits
     --------           --------   ------
     cpu                100m (2%)  100m (2%)
     memory             50Mi (1%)  50Mi (1%)
     ephemeral-storage  0 (0%)     0 (0%)
     hugepages-2Mi      0 (0%)     0 (0%)
     nvidia.com/gpu     0          0
   Events:              <none>
   ```

## Testing

部署完之后验证nvidia kubernetes device plugin

### 基本资源调度验证

1. 编写yaml文件[query.yaml](cluster_actions_yaml/device_plugin_action/query.yaml)
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
           name: query-pod
   spec:
           restartPolicy: OnFailure
           containers:
           - image: jitteam/devicequery
             name: query-ctr

             resources:
                     limits:
                             nvidia.com/gpu: 1
   ```

2. 使用命令部署上述yaml文件，观察到pod被正确调度到了worker-gpu-node1上
   
   ```
   # kubectl apply -f query.yaml
   # kubectl get pods
   NAME        READY   STATUS      RESTARTS   AGE
   query-pod   0/1     Completed   0          42s
   # kubectl describe pods query-pod
   Name:         query-pod
   Namespace:    default
   Priority:     0
   Node:         worker-gpu-node1/10.10.197.94
   Start Time:   Tue, 04 Aug 2020 15:15:36 +0800
   Labels:       <none>
   Annotations:  Status:  Succeeded
   IP:           10.88.5.10
   IPs:
     IP:  10.88.5.10
   Containers:
     query-ctr:
       Container ID:   docker://64ce6afd648a31983befc37ed7e4921ceab0b10e745f19dfaa8de785de9bef74
       Image:          jitteam/devicequery
       Image ID:       docker-pullable://jitteam/devicequery@sha256:e88bc1f5938a04ead500fd1486d95ba6340f2896b9789cacca6fd6d39a28bf09
       Port:           <none>
       Host Port:      <none>
       State:          Terminated
         Reason:       Completed
         Exit Code:    0
         Started:      Tue, 04 Aug 2020 15:15:46 +0800
         Finished:     Tue, 04 Aug 2020 15:15:46 +0800
       Ready:          False
       Restart Count:  0
       Limits:
         nvidia.com/gpu:  1
       Requests:
         nvidia.com/gpu:  1
       Environment:       <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from default-token-zhxhg (ro)
   Conditions:
     Type              Status
     Initialized       True
     Ready             False
     ContainersReady   False
     PodScheduled      True
   Volumes:
     default-token-zhxhg:
       Type:        Secret (a volume populated by a Secret)
       SecretName:  default-token-zhxhg
       Optional:    false
   QoS Class:       BestEffort
   Node-Selectors:  <none>
   Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
   Events:
     Type    Reason     Age        From                       Message
     ----    ------     ----       ----                       -------
     Normal  Scheduled  <unknown>  default-scheduler          Successfully assigned default/query-pod to worker-gpu-node1
     Normal  Pulling    9s         kubelet, worker-gpu-node1  Pulling image "jitteam/devicequery"
     Normal  Pulled     1s         kubelet, worker-gpu-node1  Successfully pulled image "jitteam/devicequery"
     Normal  Created    1s         kubelet, worker-gpu-node1  Created container query-ctr
     Normal  Started    0s         kubelet, worker-gpu-node1  Started container query-ctr
   ```

3. 使用命令获取pod的日志，成功检测底层GPU
   
   ```
   # kubectl logs query-pod
   ./deviceQuery Starting...

    CUDA Device Query (Runtime API) version (CUDART static linking)

   Detected 1 CUDA Capable device(s)

   Device 0: "NVIDIA Tegra X1"
     CUDA Driver Version / Runtime Version          10.2 / 10.0
     CUDA Capability Major/Minor version number:    5.3
     Total amount of global memory:                 3963 MBytes (4155461632 bytes)
     ( 1) Multiprocessors, (128) CUDA Cores/MP:     128 CUDA Cores
     GPU Max Clock rate:                            922 MHz (0.92 GHz)
     Memory Clock rate:                             13 Mhz
     Memory Bus Width:                              64-bit
     L2 Cache Size:                                 262144 bytes
     Maximum Texture Dimension Size (x,y,z)         1D=(65536), 2D=(65536, 65536), 3D=(4096, 4096, 4096)
     Maximum Layered 1D Texture Size, (num) layers  1D=(16384), 2048 layers
     Maximum Layered 2D Texture Size, (num) layers  2D=(16384, 16384), 2048 layers
     Total amount of constant memory:               65536 bytes
     Total amount of shared memory per block:       49152 bytes
     Total number of registers available per block: 32768
     Warp size:                                     32
     Maximum number of threads per multiprocessor:  2048
     Maximum number of threads per block:           1024
     Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
     Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
     Maximum memory pitch:                          2147483647 bytes
     Texture alignment:                             512 bytes
     Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
     Run time limit on kernels:                     Yes
     Integrated GPU sharing Host Memory:            Yes
     Support host page-locked memory mapping:       Yes
     Alignment requirement for Surfaces:            Yes
     Device has ECC support:                        Disabled
     Device supports Unified Addressing (UVA):      Yes
     Device supports Compute Preemption:            No
     Supports Cooperative Kernel Launch:            No
     Supports MultiDevice Co-op Kernel Launch:      No
     Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
     Compute Mode:
        < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

   deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.2, CUDA Runtime Version = 10.0, NumDevs = 1
   Result = PASS
   ```

### 资源竞争验证

验证nvidia kubernetes device plugin是否正确处理对同一gpu竞争

1. 编写yaml文件，[concurrency.yaml](cluster_actions_yaml/device_plugin_action/concurrency.yaml)
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod1
   spec:
     restartPolicy: OnFailure
     containers:
     - image: lxyustc.registrydomain.com:5000/nvidia/l4t-base:r32.4.3
       name: pod1-ctr
       command: ["sleep"]
       args: ["30"]

       resources:
         limits:
           nvidia.com/gpu: 1
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod2
   spec:
     restartPolicy: OnFailure
     containers:
     - image: lxyustc.registrydomain.com:5000/nvidia/l4t-base:r32.4.3
       name: pod2-ctr
       command: ["sleep"]
       args: ["30"]

       resources:
         limits:
           nvidia.com/gpu: 1
   ```

2. 使用命令部署上述yaml文件，确认可正确处理对gpu资源的竞争关系
   
   ```
   # kubectl apply -f concurrency.yaml 
   # kubectl get pods
   NAME   READY   STATUS    RESTARTS   AGE
   pod1   1/1     Running   0          12s
   pod2   0/1     Pending   0          12s
   # kubectl decribe pods pod2
   Name:         pod2
   Namespace:    default
   Priority:     0
   Node:         <none>
   Labels:       <none>
   Annotations:  Status:  Pending
   IP:
   IPs:          <none>
   Containers:
     pod2-ctr:
       Image:      lxyustc.registrydomain.com:5000/nvidia/l4t-base:r32.4.3
       Port:       <none>
       Host Port:  <none>
       Command:
         sleep
       Args:
         30
       Limits:
         nvidia.com/gpu:  1
       Requests:
         nvidia.com/gpu:  1
       Environment:       <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from default-token-zhxhg (ro)
   Conditions:
     Type           Status
     PodScheduled   False
   Volumes:
     default-token-zhxhg:
       Type:        Secret (a volume populated by a Secret)
       SecretName:  default-token-zhxhg
       Optional:    false
   QoS Class:       BestEffort
   Node-Selectors:  <none>
   Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
   Events:
     Type     Reason            Age        From               Message
     ----     ------            ----       ----               -------
     Warning  FailedScheduling  <unknown>  default-scheduler  0/5 nodes are available: 5 Insufficient nvidia.com/gpu.
     Warning  FailedScheduling  <unknown>  default-scheduler  0/5 nodes are available: 5 Insufficient nvidia.com/gpu.

   pass 30 s

   # kubectl get pods
   NAME   READY   STATUS      RESTARTS   AGE
   pod1   0/1     Completed   0          63s
   pod2   1/1     Running     0          63s

   # kubectl describe pods pod2
   Name:         pod2
   Namespace:    default
   Priority:     0
   Node:         worker-gpu-node1/10.10.197.94
   Start Time:   Tue, 04 Aug 2020 15:41:27 +0800
   Labels:       <none>
   Annotations:  Status:  Running
   IP:           10.88.5.14
   IPs:
     IP:  10.88.5.14
   Containers:
     pod2-ctr:
       Container ID:  docker://a2ddb01a68865ed423dd07d9ffaf10c8c6f83fbe331cfd62c04746a7b4a03118
       Image:         lxyustc.registrydomain.com:5000/nvidia/l4t-base:r32.4.3
       Image ID:      docker-pullable://lxyustc.registrydomain.com:5000/nvidia/l4t-base@sha256:c14913bcf6cf707581ea420c81618ec87a1e05d655495ae9559ce163d001e30b
       Port:          <none>
       Host Port:     <none>
       Command:
         sleep
       Args:
         30
       State:          Running
         Started:      Tue, 04 Aug 2020 15:41:29 +0800
       Ready:          True
       Restart Count:  0
       Limits:
         nvidia.com/gpu:  1
       Requests:
         nvidia.com/gpu:  1
       Environment:       <none>
       Mounts:
         /var/run/secrets/kubernetes.io/serviceaccount from default-token-zhxhg (ro)
   Conditions:
     Type              Status
     Initialized       True
     Ready             True
     ContainersReady   True
     PodScheduled      True
   Volumes:
     default-token-zhxhg:
       Type:        Secret (a volume populated by a Secret)
       SecretName:  default-token-zhxhg
       Optional:    false
   QoS Class:       BestEffort
   Node-Selectors:  <none>
   Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
   Events:
     Type     Reason            Age        From                       Message
     ----     ------            ----       ----                       -------
     Warning  FailedScheduling  <unknown>  default-scheduler          0/5 nodes are available: 5 Insufficient nvidia.com/gpu.
     Warning  FailedScheduling  <unknown>  default-scheduler          0/5 nodes are available: 5 Insufficient nvidia.com/gpu.
     Normal   Scheduled         <unknown>  default-scheduler          Successfully assigned default/pod2 to worker-gpu-node1
     Normal   Pulled            3s         kubelet, worker-gpu-node1  Container image "lxyustc.registrydomain.com:5000/nvidia/l4t-base:r32.4.3" already present on machine
     Normal   Created           2s         kubelet, worker-gpu-node1  Created container pod2-ctr
     Normal   Started           2s         kubelet, worker-gpu-node1  Started container pod2-ctr
   ```

## 容器运行时要求

目前nvidia官方并未推出arm64的k8s-device-plugin插件，经过修改，绕过NVML后可实现k8s-device-plugin arm64下的兼容性要求，但对于docker运行时有要求，具体如下：

|deb package|version|
|:---:|:---:|
|libnvidia-container-tools|0.9.0~beta.1|
|nvidia-container-runtime|3.1.0-1|
|nvidia-container-toolkit|1.0.1-1|
|nvidia-docker2|2.2.0-1|
