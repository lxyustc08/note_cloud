- [Nvidia Jetson Nano Config](#nvidia-jetson-nano-config)
  - [Container Runtime 配置](#container-runtime-配置)
  - [Jetson Nano 配置](#jetson-nano-配置)

# Nvidia Jetson Nano Config

## Container Runtime 配置

1. 安装nvidia-docker-runtime

```
apt install nvidia-docker-runtime
```

2. 配置docker engine，在/etc/docker/daemon.json中做如下配置

```json
"runtime": {
        "nvidia": {
         "path": "/usr/bin/nvidia-container-runtime",
         "runtimeArgs": []
     } 
}

"default-runtime": "nvidia"
```

3. 重启docker daemon

```
systemctl daemon-reload
systemctl restart docker
```

4. 验证docker正常使用GPU，如下输出Result=PASS即说明docker可正常使用nvidia GPU

```
# docker run -it jitteam/devicequery ./deviceQuery
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

## Jetson Nano 配置

1. 关闭memory swap分区，Jetson Nano使用`zram`实现内存交换分区，在/etc/systemd/文件夹下存在名称为`nvzramconfig.sh`的脚本，该脚本内容如下。该脚本在系统启动时运行，创建zram分区，将该脚本移除即可关闭swap分区

```bash
#!/bin/bash
#
# Copyright (c) 2019, NVIDIA CORPORATION.  All rights reserved.
#

NRDEVICES=$(grep -c ^processor /proc/cpuinfo | sed 's/^0$/1/')
if modinfo zram | grep -q ' zram_num_devices:' 2>/dev/null; then
        MODPROBE_ARGS="zram_num_devices=${NRDEVICES}"
elif modinfo zram | grep -q ' num_devices:' 2>/dev/null; then
        MODPROBE_ARGS="num_devices=${NRDEVICES}"
else
        exit 1
fi
modprobe zram "${MODPROBE_ARGS}"

# Calculate memory to use for zram (1/2 of ram)
totalmem=`LC_ALL=C free | grep -e "^Mem:" | sed -e 's/^Mem: *//' -e 's/  *.*//'`
mem=$((("${totalmem}" / 2 / "${NRDEVICES}") * 1024))

# initialize the devices
for i in $(seq "${NRDEVICES}"); do
        DEVNUMBER=$((i - 1))
        echo "${mem}" > /sys/block/zram"${DEVNUMBER}"/disksize
        mkswap /dev/zram"${DEVNUMBER}"
        swapon -p 5 /dev/zram"${DEVNUMBER}"
done
```

2. 配置软件源，与其他类似将所需的`kubernetes`软件源配置好即可；


