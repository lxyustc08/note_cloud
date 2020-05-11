- [Ubuntu Kernel Compile](#ubuntu-kernel-compile)
  - [Method One](#method-one)
    - [Get kernel source](#get-kernel-source)
    - [Install Dependence](#install-dependence)
    - [Edit Kernel Config](#edit-kernel-config)
    - [Compile Kernel](#compile-kernel)
    - [Install Kernel](#install-kernel)

# Ubuntu Kernel Compile

## Method One

使用dpkg打包新内核镜像

|硬件环境|硬件架构|操作系统版本|内核版本|
|:----:|:----:|:----:|:---:|
|Raspberry Pi 4|ARM64|Ubuntu 19.10|5.3.0-1023-raspi2|

### Get kernel source

1. 编辑源文件列表/etc/apt/source.list，打开源码`deb-src`源
   
   ```bash
   # See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
   # newer versions of the distribution.
   deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan main restricted
   deb-src http://mirrors.ustc.edu.cn/ubuntu-ports eoan main restricted

   ## Major bug fix updates produced after the final release of the
   ## distribution.
   deb http://mirrors.ustc.edu.cn/ubuntu-ports eoan-updates main restricted
   deb-src http://mirrors.ustc.edu.cn/ubuntu-ports eoan-updates main restricted
   ```

2. 使用`apt-get source` 获取当前内核版本源码
   
   ```terminal
   # apt-get source linux-image-$(uname -r)
   ```

### Install Dependence

1. 安装软件依赖
   ```terminal
   # apt build-dep linux linux-image-$(uname -r)
   # apt install libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf
   # apt install gcc-arm-linux-gnueabihf
   ```

   最后的gcc-arm-linux-gnueabihf为交叉编译器，此处安装的原因在于获取的内核源码包含armhf，若不安装此交叉编译器将在配置生成阶段报错。

### Edit Kernel Config

1. 进入源码文件夹，执行命令
   
   ```terminal
   # chmod a+x debian/rules
   # chmod a+x debian/scripts/*
   # chmod a+x debian/scripts/misc/*
   ```

   上述命令作用在于为相关脚本提供可执行权限

2. 生成内核配置文件
   
   ```terminal
   # fakeroot debian/rules clean
   # fakeroot debian/rules editconfigs
   ```

   第二条指令即生成内核配置文件，生成时对逐个架构进行确认是否编辑内核配置，若是，则调用`make menuconfig`对内核进行配置。

   需要开启hugetlb，在配置arm64内核时将相关选项开启，选项为2个：HugeTLB file system support，路径（File systems--->Pseudo filesystems）；HugeTLB controller，路径（General setup--->Control Group support）。配置完成后保存退出，脚本将生成配置文件`debian.raspi2/config/config.common.ubuntu`

### Compile Kernel

1. 使用debian/rules脚本构建内核镜像deb包
   
   ```terminal
   # fakeroot debian/rules clean
   # fakeroot debian/rules binary
   ```

   上述脚本先进行清理，然后构建内核镜像deb包，构建完成后，在源码上一层文件夹中出现linux-image deb

   ```terminal
   linux-image-5.3.0-1023-raspi2_5.3.0-1023.25_arm64.deb
   ```

### Install Kernel

1. 使用dpkg命令安装内核
   
   ```terminal
   # dpkg -i linux-image-5.3.0-1023-raspi2_5.3.0-1023.25_arm64.deb
   ```

2. 重启系统
   
   ```terminal
   # reboot
   ```

3. 查看此时内核版本
   
   ```terminal
   # uname -a
   Linux slave-node2-arm 5.3.0-1023-raspi2 #25 SMP Fri May 8 18:36:39 CST 2020 aarch64 aarch64 aarch64 GNU/Linux
   ```
   
   通过时间判断，内核已更新为新版本内核
