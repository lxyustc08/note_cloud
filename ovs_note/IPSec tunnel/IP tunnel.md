# IP tunnel实验
## 配置IPsec双端IP
1. node2-debian
     ```terminal
     ip_2=172.16.13.155
     ip_3=192.168.100.144
     ``` 
2. node3-debian
     ```terminal
     ip_2=172.16.13.155
     ip_3=192.168.100.144
     ```
## 配置IPsec双端网桥
1. node2-debian
   ```terminal
   # ovs-vsctl add-br br-ipsec
   # ip addr add 192.0.0.2/24 dev br-ipsec
   # ip link set br-ipsec up
   # ifconfig
   br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.244  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::5054:ff:feb0:a38d  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:b0:a3:8d  txqueuelen 1000  (Ethernet)
        RX packets 2293  bytes 134618 (131.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 739  bytes 122756 (119.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   br-ipsec: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
           inet 192.0.0.1  netmask 255.255.255.0  broadcast 0.0.0.0
           inet6 fe80::988a:fbff:fe78:ab48  prefixlen 64  scopeid 0x20<link>
           ether 9a:8a:fb:78:ab:48  txqueuelen 1000  (Ethernet)
           RX packets 0  bytes 0 (0.0 B)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 57  bytes 6638 (6.4 KiB)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
           ether 52:54:00:b0:a3:8d  txqueuelen 1000  (Ethernet)
           RX packets 2295  bytes 166846 (162.9 KiB)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 738  bytes 122654 (119.7 KiB)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ens10: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
           inet 172.16.13.155  netmask 255.255.255.0  broadcast 172.16.13.255
           inet6 fe80::5054:ff:fe5a:41a9  prefixlen 64  scopeid 0x20<link>
           ether 52:54:00:5a:41:a9  txqueuelen 1000  (Ethernet)
           RX packets 1274  bytes 80610 (78.7 KiB)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 66  bytes 7391 (7.2 KiB)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
           inet 127.0.0.1  netmask 255.0.0.0
           inet6 ::1  prefixlen 128  scopeid 0x10<host>
           loop  txqueuelen 1000  (Local Loopback)
           RX packets 8  bytes 582 (582.0 B)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 8  bytes 582 (582.0 B)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ```
2. node3-debian
    ```terminal
   # ovs-vsctl add-br br-ipsec
   # ip addr add 192.0.0.3/24 dev br-ipsec
   # ip link set br-ipsec up
   # ifconfig
   br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.22.206  netmask 255.255.255.0  broadcast 172.16.22.255
        inet6 fe80::5054:ff:fe2d:71a4  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:2d:71:a4  txqueuelen 1000  (Ethernet)
        RX packets 1740  bytes 98853 (96.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 461  bytes 59306 (57.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   br-ipsec: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
           inet 192.0.0.1  netmask 255.255.255.0  broadcast 0.0.0.0
           inet6 fe80::71:cff:fee7:de45  prefixlen 64  scopeid 0x20<link>
           ether 02:71:0c:e7:de:45  txqueuelen 1000  (Ethernet)
           RX packets 0  bytes 0 (0.0 B)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 21  bytes 2456 (2.3 KiB)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
           ether 52:54:00:2d:71:a4  txqueuelen 1000  (Ethernet)
           RX packets 1741  bytes 123265 (120.3 KiB)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 460  bytes 59204 (57.8 KiB)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ens10: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
           inet 192.168.100.144  netmask 255.255.255.0  broadcast 192.168.100.255
           inet6 fe80::5054:ff:fefa:e196  prefixlen 64  scopeid 0x20<link>
           ether 52:54:00:fa:e1:96  txqueuelen 1000  (Ethernet)
           RX packets 1149  bytes 71791 (70.1 KiB)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 68  bytes 7498 (7.3 KiB)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
           inet 127.0.0.1  netmask 255.0.0.0
           inet6 ::1  prefixlen 128  scopeid 0x10<host>
           loop  txqueuelen 1000  (Local Loopback)
           RX packets 2  bytes 78 (78.0 B)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 2  bytes 78 (78.0 B)
           TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ```
## 设置IPsec tunnel通道
有三种方式实现IPsec tunnel通道
### 使用预先定义的共享key
1. 查看默认路由  
   本次使用KVM虚拟机进行环境模拟，KVM虚拟机配置两块网卡，一块位于NAT网络中，另一块位于Route路由网络中，为保证IPSec软件包到达宿主机后可正确转发，使用Route网络网卡作为默认路由  
   + node2-debian9
        ```terminal
        # route -n
        Kernel IP routing table
       Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
       0.0.0.0         172.16.13.1     0.0.0.0         UG    0      0        0 ens10
       172.16.13.0     0.0.0.0         255.255.255.0   U     0      0        0 ens10
       192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 ens3
        ```
   + node3-debian9
        ```terminal
        # route -n
        Kernel IP routing table
         Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
         0.0.0.0         192.168.100.1   0.0.0.0         UG    0      0        0 ens10
         172.16.22.0     0.0.0.0         255.255.255.0   U     0      0        0 ens3
         192.168.100.0   0.0.0.0         255.255.255.0   U     0      0        0 ens10
        ```
2. 导入环境变量OVS_RUNDIR  
   本次测试直接使用debian系统的deb包构建的openswitch-ipsec作为ipsec管理，使用strongswan作为IKE daemon进行密钥鉴权信息交换。对于openswitch-ipsec服务而言，其调用ovs-monitor-ipsec脚本，在/var/run/openvswitch下创建pid文件，为保证ovs-monitor-ipsec工作正常，需将环境变量OVS_RUNDIR设置为/var/run/openvswitch
   ```terminal
   # export OVS_RUNDIR=/var/run/openvswitch
   # echo $OVS_RUNDIR
   /var/run/openvswitch
   ```
3. 使用ovs-ctl启动ovsdb与ovs-vswitchd
   ```terminal
   # ovs-ctl start
   [ ok ] Starting ovsdb-server.
   [FAIL] system ID not configured, please use --system-id ... failed!
   [ ok ] Configuring Open vSwitch system IDs.
   [ ok ] Starting ovs-vswitchd.
   [ ok ] Enabling remote OVSDB managers.
   # ovs-ctl status
   ovsdb-server is running with pid 1143
   ovs-vswitchd is running with pid 1217
   ```
4. 配置IPSec tunnel通道  
   ```terminal
   # ovs-vsctl add-port br-ipsec tap1
   # ovs-vsctl show
   7322faef-7f97-40cd-8e2e-17722845b3a1
    Bridge br-ipsec
        Port br-ipsec
            Interface br-ipsec
                type: internal
        Port "tap1"
            Interface "tap1"
    ovs_version: "2.11.90"
   # ovs-vsctl set interface tap1 type=gre options:remote_ip=192.168.100.144 options:psk=swordfish
   # ovs-appctl -t ovs-monitor-ipsec tunnels/show
   ```