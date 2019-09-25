# IPSec Tunnel 实验环境
## 虚拟节点拓扑
本次实验使用两台操作系统为debian 9的kvm虚拟机node2-debian9以及node3-debian9  
+ node2-debian9
  + IP地址192.168.122.244
  + 所在网络为libvirt的NAT网络，192.168.244.0/24
+ node3-debian9
  + IP地址172.16.22.206
  + 所在网络为libvirt的NAT网络，172.16.22.0/24

拓扑图如下所示  
![Alt text](IP_tunnel环境图.png "IPtunnel 实验环境图")
## 运行环境配置
本次需要安装IPsec package，步骤如下：
1. 构建debian package  
   a. 安装build-essential以及fakeroot
   ```terminal
   # apt install build-essential fakeroot
   ```
   b. 进入ovs源码包
   ```terminal
   # cd ovs
   ```
   c. 使用dpkg-checkbuilddeps检查deb包依赖
   ```terminal
   # dpkg-checkbuilddeps
   dpkg-checkbuilddeps: error: Unmet build dependencies: debhelper (>= 8) dh-autoreconf libssl-dev python-twisted-conch python-zopeinterface libunbound-dev
   ```
   d. 安装缺失依赖包
   ```terminal
   # apt install debhelper dh-autoreconf libssl-dev python-twisted-conch python-zope.interface libunbound-dev
   ```
   e. 构建deb包
   ```terminal
   # DEB_BUILD_OPTIONS='nocheck' fakeroot debian/rules binary
   ```
   构建完成后在ovs/上一级文件夹中出现deb包  


2. 安装ovs IPsec deb包
   ```terminal
   # apt install uuid-runtime
   dpkg -i openvswitch-ipsec_2.11.90-1_amd64.deb openvswitch-common_2.11.90-1_amd64.deb openvswitch-switch_2.11.90-1_amd64.deb python-openvswitch_2.11.90-1_all.deb libopenvswitch_2.11.90-1_amd64.deb
   ```
   安装完成后确认openvswitch-ipsec服务状态
   ```terminal
   # service openvswitch-ipsec status
   ● openvswitch-ipsec.service - LSB: Open vSwitch GRE-over-IPsec daemon
   Loaded: loaded (/etc/init.d/openvswitch-ipsec; generated; vendor preset: enabled)
   Active: active (running) since Thu 2019-09-05 09:16:20 HKT; 2min 22s ago
     Docs: man:systemd-sysv-generator(8)
      CPU: 2ms
   CGroup: /system.slice/openvswitch-ipsec.service
           ├─1305 /usr/lib/ipsec/starter --daemon charon
           ├─1306 /usr/lib/ipsec/charon --use-syslog
           ├─1324 /usr/bin/python2 /usr/share/openvswitch/scripts/ovs-monitor-ipsec --pidfile=/var/run/openvswitch/ovs-monitor-ipsec.pid --ike-daemon=strongswan --log-file --detach --monitor unix:/var/run/openvswitch/db.sock
           └─1325 /usr/bin/python2 /usr/share/openvswitch/scripts/ovs-monitor-ipsec --pidfile=/var/run/openvswitch/ovs-monitor-ipsec.pid --ike-daemon=strongswan --log-file --detach --monitor unix:/var/run/openvswitch/db.sock
   ```
   openvswitch-ipsec服务正常启动，说明openvswitch-ipsec安装成功