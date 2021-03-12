- [Join GPU Node AMD64 with Nvidia Using Nvidia docker](#join-gpu-node-amd64-with-nvidia-using-nvidia-docker)
  - [Prerequisites](#prerequisites)
  - [Procedure](#procedure)
- [Join GPU Node With CRI Compatible Runtime](#join-gpu-node-with-cri-compatible-runtime)
  - [Label Node](#label-node)
  - [Install Nvidia Driver](#install-nvidia-driver)
  - [部署device plugin](#部署device-plugin)

# Join GPU Node AMD64 with Nvidia Using Nvidia docker

通过nvidia-docker方式添加Nvida GPU Node方式不推荐，从Kubernetes 1.20后，Kubernetes官方不再支持docker作为其容器运行环境

## Prerequisites 

参考[此文档](../../Linux/nvidia_driver_install.md)安装nvidia驱动

## Procedure

1. 安装docker
   
   ```terminal
   $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   $ sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
   $ sudo apt install docker-ce docker-ce-cli containerd.io
   ```

2. 安装nvidia-docker2，nvidia-device-plugin目前仅支持nvidia-docker2
   
   ```terminal
   $ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   $ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
   $ curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
   $ sudo apt update
   $ sudo apt install nvidia-docker2
   ```

3. 配置docker daemon，/etc/docker/daemon.json
   
   ```json {.line-numbers} 
   {
       "default-runtime": "nvidia",
       "runtimes": {
           "nvidia": {
               "path": "/usr/bin/nvidia-container-runtime",
               "runtimeArgs": []
           }
       },
       "exec-opts": ["native.cgroupdriver=systemd"],
       "log-driver": "json-file",
       "log-opts": {
                    "max-size": "100m"
       },
       "registry-mirrors": ["https://y6h4kc1t.mirror.aliyuncs.com", "http://f1361db2.m.daocloud.io"],
       "insecure-registries" : ["lxyustc.registrydomain.com:5000"]
   }   
   ```

4. 验证`nvidia-docker2`是否安装成功
   
   ```
   $  sudo docker run --rm --gpus all lxyustc.registrydomain.com:5000/nvidia/cuda:latest nvidia-smi
   Tue Sep 15 06:39:23 2020
   +-----------------------------------------------------------------------------+
   | NVIDIA-SMI 450.66       Driver Version: 450.66       CUDA Version: 11.0     |
   |-------------------------------+----------------------+----------------------+
   | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
   | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
   |                               |                      |               MIG M. |
   |===============================+======================+======================|
   |   0  Quadro K620         Off  | 00000000:03:00.0 Off |                  N/A |
   | 36%   41C    P0     1W /  30W |      0MiB /  2000MiB |      0%      Default |
   |                               |                      |                  N/A |
   +-------------------------------+----------------------+----------------------+

   +-----------------------------------------------------------------------------+
   | Processes:                                                                  |
   |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
   |        ID   ID                                                   Usage      |
   |=============================================================================|
   |  No running processes found                                                 |
   +-----------------------------------------------------------------------------+
   ```

5. 安装Nvidia提供的device plugin

# Join GPU Node With CRI Compatible Runtime

本解决方案来源于Google GKE，经过Google GKE平台验证并商业化使用，可在Ubuntu、COS上使用，并兼容符合CRI标准的容器运行时。介绍参照[此链接](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#nvidia-gpu-device-plugin-used-by-gce)。此方案由两步骤组成：安装nvida驱动，部署device plugin。

本方法基于上述方案，修改了nVidia驱动安装使用的镜像与device plugin部署文件。具体步骤如下：

## Label Node

无论是安装Nvidia驱动亦或是安装Nvidia device plugin，均需具有Nvidia GPU的节点上进行，因此需要通过Kubernetes的label机制给Node打上标签，该标签也是后面部署时需要使用的node selector。如下所示：

```kubectl
kubectl label nodes worker-amd64-gpuceph-node1 gpu/nividia=P630

kubectl label nodes worker-amd64-gpuceph-node2 gpu/nividia=P630

kubectl label nodes worker-amd64-gpuceph-node3 gpu/nividia=P630
```

## Install Nvidia Driver

1. 获取部署所需源码：
   
   ```
   git clone https://github.com/GoogleCloudPlatform/container-engine-accelerators.git
   ```

2. 进入文件夹`container-engine-accelerators/nvidia-driver-installer/ubuntu`
3. 修改文件夹中的Dockerfile，如下所示。
   
   ```dockerfile
   FROM ubuntu:20.04


   RUN  apt-get -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false update && \
       apt-get install -y kmod gcc make curl && \
       rm -rf /var/lib/apt/lists/*

   COPY entrypoint.sh /entrypoint.sh


   CMD /entrypoint.sh
   ```

   其中`RUN  apt-get -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false update`意义为忽略时钟检查镜像仓库，为了解决可能是ubuntu  20.04镜像存在的时钟bug

4. 修改entrypoint.sh文件，修改如下内容。
   
   ```shell
   ......

   NVIDIA_DRIVER_VERSION="${NVIDIA_DRIVER_VERSION:-460.56}"

   ......

   NVIDIA_DRIVER_DOWNLOAD_URL_DEFAULT="https://us.download.nvidia.com/XFree86/Linux-x86_64/${NVIDIA_DRIVER_VERSION}/NVIDIA-Linux-x86_64-${NVIDIA_DRIVER_VERSION}.run"

   ......
   ```

   修改上述两处目的，使用460.56的Nvidia Linux驱动，以及修改下载地址

5. 构建nvidia-driver-install镜像，等待镜像构建完成，并将镜像上传至本地镜像仓库
   
   ```
   docker build -t nvidia-driver-install:20.04 .
   ```

6. 修改驱动部署文件`daemonset.yaml`，修改如下内容：
   
   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     .......
   spec:
     selector:
       .......
     updateStrategy:
       .......
     template:
       metadata:
         .......
       spec:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: gpu/nividia 
                   operator: Exists
         tolerations:
         .......
         initContainers:
         - image: "lxyustc.registrydomain.com:5000/ubuntu-nvidia-install:20.04"
           name: nvidia-driver-installer
           resources:
             .......
           securityContext:
             .......
           env:
             .......
           volumeMounts:
           .......
         containers:
         - image: "lxyustc.registrydomain.com:5000/google_containers/pause:3.2"
           name: pause
   ```

   也即修改nodeSelectorTerms中的matchExpression以及initContainers使用镜像于pause镜像。

7. 部署daemonset:
   
   ```
   kubectl apply -f daemonset.yaml
   ```

   查看各个GPU节点上的pods日志，出现如下内容表明驱动部署成功：

   ```
   +-----------------------------------------------------------------------------+
   | NVIDIA-SMI 460.56       Driver Version: 460.56       CUDA Version: 11.2     |
   |-------------------------------+----------------------+----------------------+
   | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
   | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
   |                               |                      |               MIG M. |
   |===============================+======================+======================|
   |   0  Quadro K620         Off  | 00000000:03:00.0 Off |                  N/A |
   | 33%   42C    P0     1W /  30W |      0MiB /  2000MiB |      0%      Default |
   |                               |                      |                  N/A |
   +-------------------------------+----------------------+----------------------+

   +-----------------------------------------------------------------------------+
   | Processes:                                                                  |
   |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
   |        ID   ID                                                   Usage      |
   |=============================================================================|
   |  No running processes found                                                 |
   +-----------------------------------------------------------------------------+
   Verifying Nvidia installation... DONE.
   Updating host's ld cache...
   Updating host's ld cache... DONE.
   ```

## 部署device plugin

进入路径`container-engine-accelerators/cmd/nvidia_gpu`

1. 修改Dockerfile，如下所示：
   
   ```docker file
   FROM golang:1.15 as builder
   WORKDIR /go/src/github.com/GoogleCloudPlatform/container-engine-accelerators
   COPY . .
   RUN go build cmd/nvidia_gpu/nvidia_gpu.go
   RUN chmod a+x /go/src/github.com/GoogleCloudPlatform/container-engine-accelerators/nvidia_gpu

   FROM gcr.io/distroless/base-debian10
   COPY --from=builder /go/src/github.com/GoogleCloudPlatform/container-engine-accelerators/nvidia_gpu /usr/bin/nvidia-gpu-device-plugin
   #CMD ["/usr/bin/nvidia-gpu-device-plugin", "-logtostderr"]
   # Use the CMD below to make the device plugin expose prometheus endpoint with container level GPU metrics
   CMD ["/usr/bin/nvidia-gpu-device-plugin", "-logtostderr", "-v=10", "--enable-container-gpu-metrics"]
   ```

   修改内容为取消注释最后一行，注释倒数第二行，用于pods层面上的GPU metrics导出。

2. 构建镜像，等待镜像构建完成，上传至本地镜像仓库中
   
   ```
   docker build -t nvidia-plugin:promethues .
   ```

3. 修改device plugin部署文件device-plugin.yaml，如下所示：
   
   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     ......
   spec:
     selector:
       ......
     template:
       metadata:
         ......
       spec:
         priorityClassName: system-node-critical
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: gpu/nividia
                   operator: Exists
         tolerations:
         ......
         volumes:
         ......
         containers:
         - image: "lxyustc.registrydomain.com:5000/nvidia-plugin:promethues"
           command: ["/usr/bin/nvidia-gpu-device-plugin", "-logtostderr", "--enable-container-gpu-metrics"]
           name: nvidia-gpu-device-plugin
           ports:
           ......
           env:
           ......
           resources:
            ......
           securityContext:
             ......
           volumeMounts:
           .......
     updateStrategy:
       type: RollingUpdate
   ```

   也即修改nodeSelectorTerms中的matchExpressions以及containers中的image

4. 部署device-plugin.yaml，查看部署后的device plugin pod日志，日志显示如下内容，表明device-plugin部署成功
   
   ```
   I0311 05:51:00.170991       1 nvidia_gpu.go:67] device-plugin started
   I0311 05:51:00.171091       1 nvidia_gpu.go:74] Reading GPU config file: /etc/nvidia/gpu_config.json
   I0311 05:51:00.171141       1 nvidia_gpu.go:78] Failed to parse GPU config file /etc/nvidia/gpu_config.json: unable to read gpu config file /etc/nvidia/gpu_config.json: open /etc/nvidia/gpu_config.json: no such file or directory
   I0311 05:51:00.171166       1 nvidia_gpu.go:79] Falling back to default GPU config.
   I0311 05:51:00.171173       1 nvidia_gpu.go:83] Using gpu config: {}
   I0311 05:51:07.170278       1 nvidia_gpu.go:107] Starting metrics server on port: 2112, endpoint path: /metrics, collection frequency: 30000
   I0311 05:51:07.170324       1 metrics.go:84] Starting metrics server
   I0311 05:51:07.171292       1 metrics.go:90] nvml initialized successfully. Driver version: 460.56
   I0311 05:51:07.171321       1 devices.go:105] Foud 1 GPU devices
   I0311 05:51:07.177956       1 devices.go:116] Found device nvidia0 for metrics collection
   I0311 05:51:07.178054       1 manager.go:243] will use alpha API
   I0311 05:51:07.178069       1 manager.go:257] starting device-plugin server at: /device-plugin/nvidiaGPU-1615441867.sock
   I0311 05:51:07.178350       1 manager.go:286] device-plugin server started serving
   I0311 05:51:07.271283       1 manager.go:290] falling back to v1beta1 API
   I0311 05:51:07.272605       1 manager.go:298] device-plugin registered with the kubelet
   I0311 05:51:07.273047       1 beta_plugin.go:38] device-plugin: ListAndWatch start
   I0311 05:51:07.273070       1 beta_plugin.go:133] ListAndWatch: send devices &ListAndWatchResponse{Devices:[]*Device{&Device{ID:nvidia0,Health:Healthy,Topology:nil,},},}
   ```

5. 使用kubectl describe nodes查看GPU节点可选资源，显示如下内容表明Kubernetes已可探测节点GPU资源：
   
   ```
   Capacity:
     cpu:                4
     ephemeral-storage:  102687672Ki
     hugepages-1Gi:      0
     hugepages-2Mi:      0
     memory:             16292340Ki
     nvidia.com/gpu:     1
     pods:               110
   Allocatable:
     cpu:                4
     ephemeral-storage:  94636958359
     hugepages-1Gi:      0
     hugepages-2Mi:      0
     memory:             16189940Ki
     nvidia.com/gpu:     1
     pods:               110
   ```

6. 对device plugin进行验证，编写验证文件cuda-vector-test.yaml，内容如下：
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
           name: cuda-verctor-add
   spec:
           restartPolicy: OnFailure
           containers:
                   - name: cuda-verctor-add
                     image: "lxyustc.registrydomain.com:5000/cuda-vector-add:v0.1"
                     resources:
                             limits:
                                     nvidia.com/gpu: 1
   ```

   部署该Pod，查看pod日志，显示如下内容即表明nVidia device plugin成功部署

   ```
   [Vector addition of 50000 elements]
   Copy input data from the host memory to the CUDA device
   CUDA kernel launch with 196 blocks of 256 threads
   Copy output data from the CUDA device to the host memory
   Test PASSED
   Done
   ```