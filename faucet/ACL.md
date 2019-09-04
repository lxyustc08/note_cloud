# ACL实验
## 更改faucet配置
1. 编辑faucet.yaml，内容如下：
   ```yaml
   dps:
           switch-1:
                   dp_id: 0x1
                   timeout: 7210
                   arp_neighbor_timeout: 3600
                   interfaces:
                           1:
                                   native_vlan: 100
                                   acl_in: 1
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
                   faucet_vips: ["10.100.0.254/24"]
           200:
                   faucet_vips: ["10.200.0.254/24"]

   routers:
           router-1:
                   vlans: [100, 200]
   acls:
           1:
                   - rule:
                           dl_type: 0x800
                           nw_proto: 6
                           tcp_dst: 8080
                           actions:
                                   allow: 0
                   - rule:
                           actions:
                                   allow: 1

   ```
2. 重启faucet
   ```terminal
   docker restart faucet
   ```
   查看faucet.log
   ```terminal
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 0 table config match_types: (('eth_type', False), ('in_port', False), ('ip_proto', False), ('tcp_dst', False)) name: port_acl next_tables: ['vlan', 'vip', 'eth_dst', 'flood'] output: True size: 32
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 1 table config match_types: (('eth_dst', True), ('eth_type', False), ('in_port', False), ('vlan_vid', False)) name: vlan next_tables: ['eth_src'] output: True set_fields: ('vlan_vid',) size: 32 table_id: 1 vlan_port_scale: 1.5
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 2 table config match_types: (('eth_dst', True), ('eth_src', False), ('eth_type', False), ('in_port', False), ('vlan_vid', False)) miss_goto: eth_dst name: eth_src next_tables: ['ipv4_fib', 'vip', 'eth_dst', 'flood'] output: True set_fields: ('vlan_vid', 'eth_dst') size: 32 table_id: 2 vlan_port_scale: 4.1
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 3 table config dec_ttl: True match_types: (('eth_type', False), ('ipv4_dst', True), ('vlan_vid', False)) name: ipv4_fib next_tables: ['vip', 'eth_dst', 'flood'] output: True set_fields: ('eth_dst', 'eth_src', 'vlan_vid') size: 32 table_id: 3 vlan_port_scale: 3.1
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 4 table config match_types: (('arp_tpa', False), ('eth_dst', False), ('eth_type', False), ('icmpv6_type', False), ('ip_proto', False)) name: vip next_tables: ['eth_dst', 'flood'] output: True size: 32 table_id: 4
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 5 table config exact_match: True match_types: (('eth_dst', False), ('vlan_vid', False)) miss_goto: flood name: eth_dst output: True size: 41 table_id: 5 vlan_port_scale: 4.1
   Sep 04 01:48:59 faucet.valve INFO     DPID 1 (0x1) switch-1 table ID 6 table config match_types: (('eth_dst', True), ('in_port', False), ('vlan_vid', False)) name: flood output: True size: 32 table_id: 6 vlan_port_scale: 2.1
   ```
   本次实验流表如下

<table>
    <tr>
        <th>Table ID(流表ID)</th>
        <th>Table Name(流表名称)</th>
        <th>Match Fields(匹配域)</th>
    </tr>
    <tr>
        <td>0</td>
        <td>PORT_ACL</td>
        <td>/</td>
    </tr>
    <tr>
        <td>1</td>
        <td>VLAN</td>
        <td>eth_dst, eth_type, in_port, vlan_id</td>
    </tr>
    <tr>
        <td>2</td>
        <td>ETH_SRC</td>
        <td>eth_dst, eth_src, eth_type, in_port, vlan_vid</td>
    </tr>
    <tr>
        <td>3</td>
        <td>IPV4_FIB</td>
        <td>eth_type, ipv4_dst, vlan_vid</td>
    </tr>
    <tr>
        <td>4</td>
        <td>VIP</td>
        <td>arp_tpa, eth_dst, eth_type, icmpv6_type, ip_porot</td>
    </tr>
    <tr>
        <td>5</td>
        <td>ETH_DST</td>
        <td>eth_dst, vlan_vid</td>
    </tr>
    <tr>
        <td>6</td>
        <td>FLOOD</td>
        <td>eth_dst, in_port, vlan_vid</td>
    </tr>
</table>

3. 查看此时流表
   ```terminal
   # ./dump-flow br1
    priority=20480,tcp,in_port=p1,tp_dst=8080 actions=drop
    priority=20479,in_port=p1 actions=goto_table:1
    priority=20480,in_port=p2 actions=goto_table:1
    priority=20480,in_port=p3 actions=goto_table:1
    priority=20480,in_port=p4 actions=goto_table:1
    priority=20480,in_port=p5 actions=goto_table:1
    priority=0 actions=drop
    table=1, priority=9000,in_port=p1,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:2
    table=1, priority=9000,in_port=p2,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:2
    table=1, priority=9000,in_port=p3,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4196->vlan_vid,goto_table:2
    table=1, priority=9000,in_port=p4,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4296->vlan_vid,goto_table:2
    table=1, priority=9000,in_port=p5,vlan_tci=0x0000/0x1fff actions=push_vlan:0x8100,set_field:4296->vlan_vid,goto_table:2
    table=1, priority=0 actions=drop
    table=2, priority=20490,dl_type=0x9000 actions=drop
    table=2, priority=20480,dl_src=ff:ff:ff:ff:ff:ff actions=drop
    table=2, priority=20480,dl_src=0e:00:00:00:00:01 actions=drop
    table=2, priority=16384,arp,dl_vlan=100 actions=goto_table:4
    table=2, priority=16384,arp,dl_vlan=200 actions=goto_table:4
    table=2, priority=16384,ip,dl_vlan=100,dl_dst=0e:00:00:00:00:01 actions=goto_table:3
    table=2, priority=16384,ip,dl_vlan=200,dl_dst=0e:00:00:00:00:01 actions=goto_table:3
    table=2, priority=4096,dl_vlan=100 actions=CONTROLLER:96,goto_table:5
    table=2, priority=4096,dl_vlan=200 actions=CONTROLLER:96,goto_table:5
    table=2, priority=0 actions=goto_table:5
    table=3, priority=12320,ip,dl_vlan=100,nw_dst=10.100.0.254 actions=goto_table:4
    table=3, priority=12320,ip,dl_vlan=200,nw_dst=10.200.0.254 actions=goto_table:4
    table=3, priority=12312,ip,dl_vlan=100,nw_dst=10.100.0.0/24 actions=goto_table:4
    table=3, priority=12312,ip,dl_vlan=200,nw_dst=10.100.0.0/24 actions=goto_table:4
    table=3, priority=12312,ip,dl_vlan=100,nw_dst=10.200.0.0/24 actions=goto_table:4
    table=3, priority=12312,ip,dl_vlan=200,nw_dst=10.200.0.0/24 actions=goto_table:4
    table=3, priority=0 actions=drop
    table=4, priority=12320,arp,dl_dst=ff:ff:ff:ff:ff:ff,arp_tpa=10.100.0.254 actions=CONTROLLER:64
    table=4, priority=12320,arp,dl_dst=ff:ff:ff:ff:ff:ff,arp_tpa=10.200.0.254 actions=CONTROLLER:64
    table=4, priority=12320,arp,dl_dst=0e:00:00:00:00:01 actions=CONTROLLER:64
    table=4, priority=12317,ip,dl_dst=0e:00:00:00:00:01 actions=CONTROLLER:110
    table=4, priority=12319,arp actions=goto_table:5
    table=4, priority=12316,ip actions=CONTROLLER:110,goto_table:5
    table=4, priority=12319,icmp,dl_dst=0e:00:00:00:00:01 actions=CONTROLLER:110
    table=4, priority=12318,icmp actions=CONTROLLER:110,goto_table:5
    table=4, priority=0 actions=drop
    table=5, priority=0 actions=goto_table:6
    table=6, priority=8240,dl_dst=01:00:0c:cc:cc:cc actions=drop
    table=6, priority=8240,dl_dst=01:00:0c:cc:cc:cd actions=drop
    table=6, priority=8240,dl_vlan=100,dl_dst=ff:ff:ff:ff:ff:ff actions=pop_vlan,output:p1,output:p2,output:p3
    table=6, priority=8240,dl_vlan=200,dl_dst=ff:ff:ff:ff:ff:ff actions=pop_vlan,output:p4,output:p5
    table=6, priority=8236,dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    table=6, priority=8216,dl_vlan=100,dl_dst=01:80:c2:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p1,output:p2,output:p3
    table=6, priority=8216,dl_vlan=100,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p1,output:p2,output:p3
    table=6, priority=8216,dl_vlan=200,dl_dst=01:80:c2:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p4,output:p5
    table=6, priority=8216,dl_vlan=200,dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=pop_vlan,output:p4,output:p5
    table=6, priority=8208,dl_vlan=100,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=pop_vlan,output:p1,output:p2,output:p3
    table=6, priority=8208,dl_vlan=200,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=pop_vlan,output:p4,output:p5
    table=6, priority=8192,dl_vlan=100 actions=pop_vlan,output:p1,output:p2,output:p3
    table=6, priority=8192,dl_vlan=200 actions=pop_vlan,output:p4,output:p5
    table=6, priority=0 actions=drop
   ```
   发现port_acl规则已经添加至流表中

## Tracing假想包流量跟踪
发送命令
   ```terminal
   # ovs-appctl ofproto/trace br1 in_port=p1,dl_src=00:01:02:03:04:05,dl_dst=0e:00:00:00:00:01,tcp,nw_src=10.100.0.1,nw_dst=10.200.0.1,nw_ttl=64,tp_dst=80 -generate
   Flow: tcp,in_port=1,vlan_tci=0x0000,dl_src=00:01:02:03:04:05,dl_dst=0e:00:00:00:00:01,nw_src=10.100.0.1,nw_dst=10.200.0.1,nw_tos=0,nw_ecn=0,nw_ttl=64,tp_src=0,tp_dst=80,tcp_flags=0

   bridge("br1")
   -------------
    1. in_port=1, priority 20479, cookie 0x5adc15c0
       goto_table:1
    2. in_port=1,vlan_tci=0x0000/0x1fff, priority 9000, cookie 0x5adc15c0
       push_vlan:0x8100
       set_field:4196->vlan_vid
       goto_table:2
    3. ip,dl_vlan=100,dl_dst=0e:00:00:00:00:01, priority 16384, cookie 0x5adc15c0
       goto_table:3
    4. ip,dl_vlan=100,nw_dst=10.200.0.0/24, priority 12312, cookie 0x5adc15c0
       goto_table:4
    5. ip,dl_dst=0e:00:00:00:00:01, priority 12317, cookie 0x5adc15c0
       CONTROLLER:110

   Final flow: tcp,in_port=1,dl_vlan=100,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=00:01:02:03:04:05,dl_dst=0e:00:00:00:00:01,nw_src=10.100.0.1,nw_dst=10.200.0.1,nw_tos=0,nw_ecn=0,nw_ttl=64,tp_src=0,tp_dst=80,tcp_flags=0
   Megaflow: recirc_id=0,eth,tcp,in_port=1,vlan_tci=0x0000/0x1fff,dl_src=00:01:02:03:04:05,dl_dst=0e:00:00:00:00:01,nw_dst=10.200.0.0/25,nw_frag=no,tp_dst=0x0/0xf000
   Datapath actions: push_vlan(vid=100,pcp=0),userspace(pid=3930819907,controller(reason=1,dont_send=0,continuation=0,recirc_id=4,rule_cookie=0x5adc15c0,controller_id=0,max_len=110))
   ```
注意查看命令的输出，其生成的MegaFlow如下：
```terminal
Megaflow: recirc_id=0,eth,tcp,in_port=1,vlan_tci=0x0000/0x1fff,dl_src=00:01:02:03:04:05,dl_dst=0e:00:00:00:00:01,nw_dst=10.200.0.0/25,nw_frag=no,tp_dst=0x0/0xf000
```
其中tp_dst匹配规则为0x0/0xf000
原因如下：
> 80的十六进制为0x0000 0000 0101 0000  
> 8080的十六进制为0x0001 1111 1001 0000  
> **两者最关键的不同在于第13位，通过第13为的不同ovs即可分辨出8080和80，通过这种生成方式，匹配种类最多只有16种，而如果单纯使用端口匹配则有65536种端口，增加MegaFlow空间大小**