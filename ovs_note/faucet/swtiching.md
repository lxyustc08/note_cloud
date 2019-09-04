# Switching实验
模拟二层网络设备的mac数据包交换
## 配置faucet
编辑~/WorkSpace/dockeretc/faucet/faucet.yaml
```yaml
dps:
        switch-1:
                dp_id: 0x1
                timeout: 7210
                arp_neighbor_timeout: 3600
                interfaces:
                        1:
                                native_vlan: 100
                        2:
                                native_vlan: 100
                        3:
                                native_vlan: 100
                        4:
                                native_vlan: 200
                        5:
                                native_vlan: 200
vlans:
        100:
        200:
```
重启faucet容器
```terminal
docker restart faucet
```
查看faucet.log，内容如下：
```terminal
Sep 03 01:45:56 faucet INFO     Reloading configuration
Sep 03 01:45:56 faucet INFO     configuration /etc/faucet/faucet.yaml changed, analyzing differences
Sep 03 01:45:56 faucet INFO     Add new datapath DPID 1 (0x1)
Sep 03 01:45:56 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 0 table config match_types: (('eth_dst', True), ('eth_type', False), ('in_port', False), ('vlan_vid', False)) name: vlan next_tables: ['eth_src'] output: True set_fields: ('vlan_vid',) size: 32 vlan_port_scale: 1.5
Sep 03 01:45:56 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 1 table config match_types: (('eth_dst', True), ('eth_src', False), ('eth_type', False), ('in_port', False), ('vlan_vid', False)) miss_goto: eth_dst name: eth_src next_tables: ['eth_dst', 'flood'] output: True set_fields: ('vlan_vid', 'eth_dst') size: 32 table_id: 1 vlan_port_scale: 4.1
Sep 03 01:45:56 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 2 table config exact_match: True match_types: (('eth_dst', False), ('vlan_vid', False)) miss_goto: flood name: eth_dst output: True size: 41 table_id: 2 vlan_port_scale: 4.1
Sep 03 01:45:56 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 3 table config match_types: (('eth_dst', True), ('in_port', False), ('vlan_vid', False)) name: flood output: True size: 32 table_id: 3 vlan_port_scale: 2.1
```
**本次测试的流表情况**
<table>
    <tr>
        <th>Table ID（流表ID）</th>
        <th>Table Name(流表名称)</th>
        <th>Match fields(匹配域)</th>
    </tr>
    <tr>
        <td>0</td>
        <td>VLAN</td>
        <td>eth_dst, eth_type, in_port, vlan_id</td>
    </tr>
    <tr>
        <td>1</td>
        <td>ETH_SRC</td>
        <td>eth_dst, eth_src, eth_type, in_port, vlan_id</td>
    </tr>
    <tr>
        <td>2</td>
        <td>ETH_DST</td>
        <td>eth_dst, vlan_vid</td>
    </tr>
    <tr>
        <td>3</td>
        <td>FLOOD</td>
        <td>eth_dst, in_port, vlan_id</td>
    </tr>
</table>

## 配置网桥br1
### Step1. 启动ovs-vswithd与ovsdb-server
使用ovs-ctl启动ovs-vswitchd与ovsdb-server
```terminal
# ovs-ctl start
[ ok ] Starting ovsdb-server.
[FAIL] system ID not configured, please use --system-id ... failed!
[ ok ] Configuring Open vSwitch system IDs.
[ ok ] Starting ovs-vswitchd.
[ ok ] Enabling remote OVSDB managers.
```
查看ovs-vswitchd与ovsdb-server的运行状态
```terminal
# ovs-ctl status
ovsdb-server is running with pid 6281
ovs-vswitchd is running with pid 6355
```

### Step2. 创建并网桥br1
1. 使用ovs-vsctl创建网桥
   ```terminal
   # ovs-vsctl add-br br1
   # ovs-vsctl show
   d961dc9f-0c4c-4270-9875-0e9c0c18ef61
       Bridge "br1"
           Port "br1"
               Interface "br1"
                   type: internal
       ovs_version: "2.11.90"
   ```
2. 将模拟端口p1~p5加入到网桥br1中
   ```terminal
   # for i in `seq 1 5`; do ovs-vsctl add-port br1 p${i}; done
   # ovs-vsctl show
   d961dc9f-0c4c-4270-9875-0e9c0c18ef61
    Bridge "br1"
        Port "p4"
            Interface "p4"
        Port "p5"
            Interface "p5"
        Port "p3"
            Interface "p3"
        Port "p1"
            Interface "p1"
        Port "br1"
            Interface "br1"
                type: internal
        Port "p2"
            Interface "p2"
    ovs_version: "2.11.90"
   ```
3. 指定br1的配置文件
   ```terminal
   # ovs-vsctl set bridge br1 other-config:datapath-id=0x1
   # ovs-vsctl list bridge
   _uuid               : 7174e323-c9fe-4aab-9a47-c8ed8755ec7b
   auto_attach         : []
   controller          : []
   datapath_id         : "000022e37471ab4a"
   datapath_type       : ""
   datapath_version    : "2.11.90"
   external_ids        : {}
   fail_mode           : []
   flood_vlans         : []
   flow_tables         : {}
   ipfix               : []
   mcast_snooping_enable: false
   mirrors             : []
   name                : "br1"
   netflow             : []
   other_config        : {datapath-id="0x1"}
   ports               : [2489612d-2bb3-4e01-8258-d9db0d057276, 3a855e42-9e8e-4c9d-934c-93a99da2aa58, 4386f25b-5214-46f1-aea1-256bb1ab3e2d, 4c0355e1-0c32-42cc-9343-548241b47244, 7351bc5c-d399-4246-958f-788e6bd32903, d4afe793-53b4-4294-ade1-d8f957ee5276]
   protocols           : []
   rstp_enable         : false
   rstp_status         : {}
   sflow               : []
   status              : {}
   stp_enable          : false
   ```

4. 设置p1~p5，将其配置指向外部配置
   ```terminal
   # for i in `seq 1 5`; do ovs-vsctl set interface p${i} ofport_request=${i}; done
   # for i in `seq 1 5`; do ovs-vsctl get interface p${i} ofport; done
   1
   2
   3
   4
   5
   ```
5. 将br1连接至控制器faucet，也即运行在宿主机6653端口的faucet
   ```terminal
   # ovs-vsctl set-controller br1 tcp:192.168.122.1:6653
   ```
   在faucet所在的宿主机中查看faucet运行日志
   ```terminal
   $ cat faucet.log
   Sep 03 02:42:14 faucet.valve INFO     DPID 1 (0x1) switch-1 Cold start configuring DP
   Sep 03 02:42:14 faucet.valve INFO     DPID 1 (0x1) switch-1 Configuring VLAN 100 vid:100 untagged: Port 1,Port 2,Port 3
   Sep 03 02:42:14 faucet.valve INFO     DPID 1 (0x1) switch-1 Configuring VLAN 200 vid:200 untagged: Port 4,Port 5
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 1 up status False reason ADD state 1
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 1 (1) up
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 2 up status False reason ADD state 1
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 2 (2) up
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 3 up status False reason ADD state 1
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 3 (3) up
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 4 up status False reason ADD state 1
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 4 (4) up
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 5 up status False reason ADD state 1
   Sep 03 02:53:28 faucet.valve INFO     DPID 1 (0x1) switch-1 Port 5 (5) up
   ```
6. 设置控制器模式为带外模式out-of-band
   ```terminal
   # ovs-vsctl set controller br1 connection-mode=out-of-band
   # ovs-vsctl list controller
   _uuid               : 8c411a1c-7281-44df-80fd-9aad5d4c418c
   connection_mode     : out-of-band
   controller_burst_limit: []
   controller_rate_limit: []
   enable_async_messages: []
   external_ids        : {}
   inactivity_probe    : []
   is_connected        : true
   local_gateway       : []
   local_ip            : []
   local_netmask       : []
   max_backoff         : []
   other_config        : {}
   role                : other
   status              : {sec_since_connect="948", state=ACTIVE}
   target              : "tcp:192.168.122.1:6653"
   type                : []
   ```
7. 查看此时的流表
   ```terminal
   # ./dumps-flows br1
    priority=9000,in_port=p1,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:1
    priority=9000,in_port=p2,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:1
    priority=9000,in_port=p3,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:1
    priority=9000,in_port=p4,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4296->vlan_vid,goto_table:1
    priority=9000,in_port=p5,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4296->vlan_vid,goto_table:1
    priority=0 actions=drop
    table=1, priority=20490,dl_type=0x9000 actions=drop
    table=1, priority=20480,dl_src=ff:ff:ff:ff:ff:ff actions=drop
    table=1, priority=20480,dl_src=0e:00:00:00:00:01 actions=drop
    table=1, priority=4096,dl_vlan=100 actions=CONTROLLER:96,goto_table:2
    table=1, priority=4096,dl_vlan=200 actions=CONTROLLER:96,goto_table:2
    table=1, priority=0 actions=goto_table:2
    table=2, priority=0 actions=goto_table:3
    table=3, priority=8240,dl_dst=01:00:0c:cc:cc:cc actions=drop
    table=3, priority=8240,dl_dst=01:00:0c:cc:cc:cd actions=drop
    table=3, priority=8240,dl_vlan=100,dl_dst=ff:ff:ff:ff:ff:ff actions=pop_vlan,output:p1,output:p2,output:p3
    table=3, priority=8240,dl_vlan=200,dl_dst=ff:ff:ff:ff:ff:ff actions=pop_vlan,output:p4,output:p5
    table=3, priority=8236,dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    table=3, priority=8216,dl_vlan=100,dl_dst=01:80:c2:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p1,output:p2,output:p3
    table=3, priority=8216,dl_vlan=100,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p1,output:p2,output:p3
    table=3, priority=8216,dl_vlan=200,dl_dst=01:80:c2:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p4,output:p5
    table=3, priority=8216,dl_vlan=200,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p4,output:p5
    table=3, priority=8208,dl_vlan=100,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=pop_vlan,output:p1,output:p2,output:p3
    table=3, priority=8208,dl_vlan=200,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=pop_vlan,output:p4,output:p5
    table=3, priority=8192,dl_vlan=100 actions=pop_vlan,output:p1,output:p2,output:p3
    table=3, priority=8192,dl_vlan=200 actions=pop_vlan,output:p4,output:p5
    table=3, priority=0 actions=drop
   ```

## Tracing 流量跟踪模拟
使用ofproto/trace开展流量跟踪模拟实验
### Step1. 最简单的流量跟踪模拟实验
1. 输入命令
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=p1
   Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

   bridge("br1")
   -------------
    1. in_port=1,vlan_tci=0x0000/0x1fff, priority 9000, cookie 0x5adc15c0
       push_vlan:0x8100
       set_field:4196->vlan_vid
       goto_table:1
    2. dl_vlan=100, priority 4096, cookie 0x5adc15c0
       CONTROLLER:96
       goto_table:2
    3. priority 0, cookie 0x5adc15c0
       goto_table:3
    4. dl_vlan=100, priority 8192, cookie 0x5adc15c0
       pop_vlan
       output:1
        >> skipping output to input port
       output:2
       output:3

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000
   Datapath actions: push_vlan(vid=100,pcp=0),userspace(pid=3736216755,controller(reason=1,dont_send=1,continuation=0,recirc_id=1,rule_cookie=0x5adc15c0,controller_id=0,max_len=96)),pop_vlan,3,4
   ```

### Step2. trigger mac learning，触发mac地址学习
1. **保存原始流量**
   ```terminal
   # ./save-flows br1 > saveflows1
   # cat saveflows1
    cookie=0x5adc15c0, duration=1801.782s, table=0, n_packets=0, n_bytes=0, priority=0 actions=drop
    cookie=0x5adc15c0, duration=1801.782s, table=1, n_packets=0, n_bytes=0, priority=0 actions=goto_table:2
    cookie=0x5adc15c0, duration=1801.782s, table=2, n_packets=0, n_bytes=0, priority=0 actions=goto_table:3
    cookie=0x5adc15c0, duration=1801.782s, table=3, n_packets=0, n_bytes=0, priority=0 actions=drop
    cookie=0x5adc15c0, duration=1801.782s, table=1, n_packets=0, n_bytes=0, priority=4096,dl_vlan=100 actions=CONTROLLER:96,goto_table:2
    cookie=0x5adc15c0, duration=1801.782s, table=1, n_packets=0, n_bytes=0, priority=4096,dl_vlan=200 actions=CONTROLLER:96,goto_table:2
    cookie=0x5adc15c0, duration=1128.086s, table=3, n_packets=0, n_bytes=0, priority=8192,dl_vlan=100 actions=pop_vlan,output:1,output:2,output:3
    cookie=0x5adc15c0, duration=1128.084s, table=3, n_packets=0, n_bytes=0, priority=8192,dl_vlan=200 actions=pop_vlan,output:4,output:5
    cookie=0x5adc15c0, duration=1128.086s, table=3, n_packets=0, n_bytes=0, priority=8208,dl_vlan=100,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=pop_vlan,output:1,output:2,output:3
    cookie=0x5adc15c0, duration=1128.084s, table=3, n_packets=0, n_bytes=0, priority=8208,dl_vlan=200,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=pop_vlan,output:4,output:5
    cookie=0x5adc15c0, duration=1128.086s, table=3, n_packets=0, n_bytes=0, priority=8216,dl_vlan=100,dl_dst=01:80:c2:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:1,output:2,output:3
    cookie=0x5adc15c0, duration=1128.086s, table=3, n_packets=0, n_bytes=0, priority=8216,dl_vlan=100,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:1,output:2,output:3
    cookie=0x5adc15c0, duration=1128.084s, table=3, n_packets=0, n_bytes=0, priority=8216,dl_vlan=200,dl_dst=01:80:c2:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:4,output:5
    cookie=0x5adc15c0, duration=1128.084s, table=3, n_packets=0, n_bytes=0, priority=8216,dl_vlan=200,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:4,output:5
    cookie=0x5adc15c0, duration=1801.783s, table=3, n_packets=0, n_bytes=0, priority=8236,dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    cookie=0x5adc15c0, duration=1801.783s, table=3, n_packets=0, n_bytes=0, priority=8240,dl_dst=01:00:0c:cc:cc:cc actions=drop
    cookie=0x5adc15c0, duration=1801.783s, table=3, n_packets=0, n_bytes=0, priority=8240,dl_dst=01:00:0c:cc:cc:cd actions=drop
    cookie=0x5adc15c0, duration=1128.086s, table=3, n_packets=0, n_bytes=0, priority=8240,dl_vlan=100,dl_dst=ff:ff:ff:ff:ff:ff actions=pop_vlan,output:1,output:2,output:3
    cookie=0x5adc15c0, duration=1128.084s, table=3, n_packets=0, n_bytes=0, priority=8240,dl_vlan=200,dl_dst=ff:ff:ff:ff:ff:ff actions=pop_vlan,output:4,output:5
    cookie=0x5adc15c0, duration=1128.146s, table=0, n_packets=0, n_bytes=0, priority=9000,in_port=1,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:1
    cookie=0x5adc15c0, duration=1128.110s, table=0, n_packets=0, n_bytes=0, priority=9000,in_port=2,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:1
    cookie=0x5adc15c0, duration=1128.108s, table=0, n_packets=0, n_bytes=0, priority=9000,in_port=3,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:1
    cookie=0x5adc15c0, duration=1128.086s, table=0, n_packets=0, n_bytes=0, priority=9000,in_port=4,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4296->vlan_vid,goto_table:1
    cookie=0x5adc15c0, duration=1128.084s, table=0, n_packets=0, n_bytes=0, priority=9000,in_port=5,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4296->vlan_vid,goto_table:1
    cookie=0x5adc15c0, duration=1801.783s, table=1, n_packets=0, n_bytes=0, priority=20480,dl_src=ff:ff:ff:ff:ff:ff actions=drop
    cookie=0x5adc15c0, duration=1801.783s, table=1, n_packets=0, n_bytes=0, priority=20480,dl_src=0e:00:00:00:00:01 actions=drop
    cookie=0x5adc15c0, duration=1801.783s, table=1, n_packets=0, n_bytes=0, priority=20490,dl_type=0x9000 actions=drop
   ```
2. **使用ofproto/trace发送二层网络数据包**
   ```terminal
   # ovs-appctl ofproto/trace br1 in_port=p1,dl_src=00:11:11:00:00:00,dl_dst=00:22:22:00:00:00 -generate
   Flow: in_port=1,vlan_tci=0x0000,dl_src=00:11:11:00:00:00,dl_dst=00:22:22:00:00:00,dl_type=0x0000

   bridge("br1")
   -------------
    1. in_port=1,vlan_tci=0x0000/0x1fff, priority 9000, cookie 0x5adc15c0
       push_vlan:0x8100
       set_field:4196->vlan_vid
       goto_table:1
    2. dl_vlan=100, priority 4096, cookie 0x5adc15c0
       CONTROLLER:96
       goto_table:2
    3. priority 0, cookie 0x5adc15c0
       goto_table:3
    4. dl_vlan=100, priority 8192, cookie 0x5adc15c0
       pop_vlan
       output:1
        >> skipping output to input port
       output:2
       output:3

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,vlan_tci=0x0000,dl_src=00:11:11:00:00:00,dl_dst=00:22:22:00:00:00,dl_type=0x0000
   Datapath actions: push_vlan(vid=100,pcp=0),userspace(pid=3736216755,controller(reason=1,dont_send=0,continuation=0,recirc_id=2,rule_cookie=0x5adc15c0,controller_id=0,max_len=96)),pop_vlan,3,4
   ```
   发现按照原始流表规定进行处理
3. **查看faucet.log**
   ```terminal
   Sep 03 03:14:15 faucet.valve INFO     DPID 1 (0x1) switch-1 L2 learned 00:11:11:00:00:00 (L2 type 0x0000, L2 dst 00:22:22:00:00:00, L3 src None, L3 dst None) Port 1 VLAN 100 (1 hosts total)
   ```
   faucet学习了p1端口上的mac地址
4. **与原始流表进行比较**
   ```terminal
   # ./diff-flows saveflows1 br1
   +table=1 priority=8191,in_port=1,dl_vlan=100,dl_src=00:11:11:00:00:00 hard_timeout=7105 actions=goto_table:2
   +table=2 priority=8192,dl_vlan=100,dl_dst=00:11:11:00:00:00 idle_timeout=10710 actions=pop_vlan,output:1
   ```
   变化情况如下：
   + ETH_SRC中增加源地址为00:11:11:00:00:00的匹配规则
   + ETH_DST中增加目的地址为00:11:11:00:00:00的匹配规则
5. **使用ofproto/trace，从p2发送数据包至p1**
   ```terminal
   # ovs-appctl ofproto/trace br1 in_port=p2,dl_src=00:22:22:00:00:00,dl_dst=00:11:11:00:00:00 -generate
   Flow: in_port=2,vlan_tci=0x0000,dl_src=00:22:22:00:00:00,dl_dst=00:11:11:00:00:00,dl_type=0x0000

   bridge("br1")
   -------------
    1. in_port=2,vlan_tci=0x0000/0x1fff, priority 9000, cookie 0x5adc15c0
       push_vlan:0x8100
       set_field:4196->vlan_vid
       goto_table:1
    2. dl_vlan=100, priority 4096, cookie 0x5adc15c0
       CONTROLLER:96
       goto_table:2
    3. dl_vlan=100,dl_dst=00:11:11:00:00:00, priority 8192, cookie 0x5adc15c0
       pop_vlan
       output:1

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=2,vlan_tci=0x0000,dl_src=00:22:22:00:00:00,dl_dst=00:11:11:00:00:00,dl_type=0x0000
   Datapath actions: push_vlan(vid=100,pcp=0),userspace(pid=3461266232,controller(reason=1,dont_send=0,continuation=0,recirc_id=3,rule_cookie=0x5adc15c0,controller_id=0,max_len=96)),pop_vlan,2
   ```
   查看faucet.log
   ```terminal
   Sep 03 03:22:26 faucet.valve INFO     DPID 1 (0x1) switch-1 L2 learned 00:22:22:00:00:00 (L2 type 0x0000, L2 dst 00:11:11:00:00:00, L3 src None, L3 dst None) Port 2 VLAN 100 (2 hosts total)
   ```
   faucet中学习了p2上端口上连接的mac地址
6. **与原始流表进行比较**
   ```terminal
   +table=1 priority=8191,in_port=1,dl_vlan=100,dl_src=00:11:11:00:00:00 hard_timeout=7105 actions=goto_table:2
   +table=1 priority=8191,in_port=2,dl_vlan=100,dl_src=00:22:22:00:00:00 hard_timeout=7193 actions=goto_table:2
   +table=2 priority=8192,dl_vlan=100,dl_dst=00:11:11:00:00:00 idle_timeout=10710 actions=pop_vlan,output:1
   +table=2 priority=8192,dl_vlan=100,dl_dst=00:22:22:00:00:00 idle_timeout=10798 actions=pop_vlan,output:2
   ```
   与步骤4相比变化情况如下：
   + ETH_SRC中添加源地址为00:22:22:00:00:00的匹配规则
   + ETH_DST中添加目的地址为00:22:22:00:00:00的匹配规则
7. **此时重复发送步骤2与步骤5的命令**
   ```terminal
   # ovs-appctl ofproto/trace br1 in_port=p1,dl_src=00:11:11:00:00:00,dl_dst=00:22:22:00:00:00 -generate
   Flow: in_port=1,vlan_tci=0x0000,dl_src=00:11:11:00:00:00,dl_dst=00:22:22:00:00:00,dl_type=0x0000

   bridge("br1")
   -------------
    1. in_port=1,vlan_tci=0x0000/0x1fff, priority 9000, cookie 0x5adc15c0
       push_vlan:0x8100
       set_field:4196->vlan_vid
       goto_table:1
    2. in_port=1,dl_vlan=100,dl_src=00:11:11:00:00:00, priority 8191, cookie 0x5adc15c0
       goto_table:2
    3. dl_vlan=100,dl_dst=00:22:22:00:00:00, priority 8192, cookie 0x5adc15c0
       pop_vlan
       output:2

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,vlan_tci=0x0000/0x1fff,dl_src=00:11:11:00:00:00,dl_dst=00:22:22:00:00:00,dl_type=0x0000
   Datapath actions: 3

   # ovs-appctl ofproto/trace br1 in_port=p2,dl_src=00:22:22:00:00:00,dl_dst=00:11:11:00:00:00 -generate
   Flow: in_port=2,vlan_tci=0x0000,dl_src=00:22:22:00:00:00,dl_dst=00:11:11:00:00:00,dl_type=0x0000

   bridge("br1")
   -------------
    1. in_port=2,vlan_tci=0x0000/0x1fff, priority 9000, cookie 0x5adc15c0
       push_vlan:0x8100
       set_field:4196->vlan_vid
       goto_table:1
    2. in_port=2,dl_vlan=100,dl_src=00:22:22:00:00:00, priority 8191, cookie 0x5adc15c0
       goto_table:2
    3. dl_vlan=100,dl_dst=00:11:11:00:00:00, priority 8192, cookie 0x5adc15c0
       pop_vlan
       output:1

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=2,vlan_tci=0x0000/0x1fff,dl_src=00:22:22:00:00:00,dl_dst=00:11:11:00:00:00,dl_type=0x0000
   Datapath actions: 2
   ```
   可发现无需连接控制器，直接按照本地流表规则进行数据包传递