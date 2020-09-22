# NVIDIA DRIVER INSTALL

Linux下Nvidia驱动有多种方式

## Ubuntu Driver

***

对于Ubuntu发行版而言，可使用Ubuntu Driver安装相关驱动

> **不建议Ubuntu Server采用此种方式进行**

1. 安装Ubuntu Driver package
   
   ```
   $ sudo apt install ubuntu-driver-common
   ```

2. 查看当前可安装的驱动
   
   ```
   $ sudo ubuntu-drivers devices
   ```

3. 选择特定的安装包安装
   
   ```
   $ sudo apt install nvidia-driver-440
   ```

4. 禁用开源驱动nouveau
   
   ```
   $ sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
   $ sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
   $ sudo update-initramfs -u
   ```

## Nvidia 官方脚本

***

1. 前往nvidia官方网址 https://www.nvidia.com/Download/index.aspx 查找设备型号对应的驱动；

2. 禁用开源驱动nouveau，同上处理

3. 安装必要依赖，主要包括gcc，pkg-config，make
   
   ```
   $ sudo apt install gcc
   $ sudo apt install pkg-config
   $ sudo apt install make
   ```

4. 运行安装脚本
   
   ```
   $ sudo bash NVIDIA-Linux-x86_64-450.66.run
   ```
