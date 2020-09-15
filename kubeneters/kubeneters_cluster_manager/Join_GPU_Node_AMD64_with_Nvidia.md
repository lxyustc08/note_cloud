- [Join GPU Node AMD64 with Nvidia](#join-gpu-node-amd64-with-nvidia)
  - [Prerequisites](#prerequisites)
  - [Procedure](#procedure)

# Join GPU Node AMD64 with Nvidia

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
