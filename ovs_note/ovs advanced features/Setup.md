# Setup初始化scenario
## step1
1. 创建网桥br0
   ```terminal
   # ovs-vsctl add-br br0 -- set Bridge br0 fail_mode=secure
   # ovs-vsctl show
   7322faef-7f97-40cd-8e2e-17722845b3a1
       Bridge "br0"
           fail_mode: secure
           Port "br0"
               Interface "br0"
                   type: internal
       ovs_version: "2.11.90"
   ```
2. 添加p1\~p4端口至br0，并将其open flow端口设置为1\~4
   ```terminal
   for i in `seq 1 4`
   do
        ovs-vsctl add-port br0 p$i -- set interface p$i ofport_request=$i
   done
   # ovs-vsctl show
   7322faef-7f97-40cd-8e2e-17722845b3a1
    Bridge "br0"
        fail_mode: secure
        Port "br0"
            Interface "br0"
                type: internal
        Port "p2"
            Interface "p2"
        Port "p4"
            Interface "p4"
        Port "p1"
            Interface "p1"
        Port "p3"
            Interface "p3"
    ovs_version: "2.11.90"
   # ovs-ofctl show br0
   OFPT_FEATURES_REPLY (xid=0x2): dpid:0000c2b143c58f4c
   n_tables:254, n_buffers:0
   capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
   actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
    1(p1): addr:12:7a:bd:7d:3e:89
        config:     PORT_DOWN
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    2(p2): addr:46:c8:df:43:83:63
        config:     PORT_DOWN
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    3(p3): addr:92:88:89:f2:dc:df
        config:     PORT_DOWN
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    4(p4): addr:9a:9a:cd:3e:6b:29
        config:     PORT_DOWN
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    LOCAL(br0): addr:c2:b1:43:c5:8f:4c
        config:     PORT_DOWN
        state:      LINK_DOWN
        speed: 0 Mbps now, 0 Mbps max
   OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
   ```
3. 使用ovs-ofctl命令将各端口up
   ```terminal
   for i in `seq 1 4`
   do
        ovs-ofctl mod-port br0 p$i up
   done
   # ovs-ofctl show br0
   OFPT_FEATURES_REPLY (xid=0x2): dpid:0000c2b143c58f4c
   n_tables:254, n_buffers:0
   capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
   actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
    1(p1): addr:12:7a:bd:7d:3e:89
        config:     0
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    2(p2): addr:46:c8:df:43:83:63
        config:     0
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    3(p3): addr:92:88:89:f2:dc:df
        config:     0
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    4(p4): addr:9a:9a:cd:3e:6b:29
        config:     0
        state:      LINK_DOWN
        current:    10MB-FD COPPER
        speed: 10 Mbps now, 0 Mbps max
    LOCAL(br0): addr:c2:b1:43:c5:8f:4c
        config:     PORT_DOWN
        state:      LINK_DOWN
        speed: 0 Mbps now, 0 Mbps max
   OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
   ```

## step 2 Implemting Table 0: Adimission control
Table 0是包进入switch的第一个流表，在这个阶段可基于一定规则将包丢弃。
1. 添加广播包过滤规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00, actions=drop"
   ```
2. 添加IEEE 802.1D 生成树协议过滤规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0, actions=drop"
   ```
3. 添加默认包处理规则
   ```terminal
   # ovs-ofctl add-flow br0 "table 0, priority=0, actions=resubmit(,1)"
   ```
   > resubmit是Open vSwitch对OpenFlow的扩展
4. 查看此时流表
   ```terminal
   # ovs-ofctl dump-flows br0
    cookie=0x0, duration=478.801s, table=0, n_packets=0, n_bytes=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
    cookie=0x0, duration=191.987s, table=0, n_packets=0, n_bytes=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    cookie=0x0, duration=67.984s, table=0, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,1)

   ```
5. 验证Table 0
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=1,dl_dst=01:80:c2:00:00:05
   Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=01:80:c2:00:00:05,dl_type=0x0000

   bridge("br0")
   -------------
    1. dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0, priority 32768
       drop

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop

   # ovs-appctl ofproto/trace br0 in_port=1,dl_dst=01:80:c2:00:00:10
   Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=01:80:c2:00:00:10,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. No match.
       drop

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=01:80:c2:00:00:10/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop
   ```
## step 3 Impletmenting Table1: VLAN Input Processing
The purpose of table 1 is validate the packet’s VLAN, based on the VLAN configuration of the switch port through which the packet entered the switch. We will also use it to attach a VLAN header to packets that arrive on an access port, which allows later processing stages to rely on the packet’s VLAN always being part of the VLAN header, reducing special cases.
1. 添加默认丢弃规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=1, priority=0, actions=drop"
   ```
2. 添加汇聚端口trunk port规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=1, priority=99, in_port=1, actions=resubmit(,2)"
   ```
   > 对于trunk port而言，其负责运送所有流量无论是否是tag packet，因此只需要将包往下一个表传即可
3. 添加access port规则
   ```terminal
   ovs-ofctl add-flows br0 - <<'EOF'
   table=1, priority=99, in_port=2, vlan_tci=0, actions=mod_vlan_vid:20, resubmit(,2)
   table=1, priority=99, in_port=3, vlan_tci=0, actions=mod_vlan_vid:30, resubmit(,2)
   table=1, priority=99, in_port=4, vlan_tci=0, actions=mod_vlan_vid:30, resubmit(,2)
   EOF
   ```
   > 对于access port而言，规则接收所有untag packet，并打上vlan_id头，将其送至下一个表
   >> *注：此处不含处理任何带有802.1Q的包规则*
4. 查看此时流表
   ```terminal
   # ovs-ofctl dump-flows br0
    cookie=0x0, duration=20512.470s, table=0, n_packets=0, n_bytes=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
    cookie=0x0, duration=20225.657s, table=0, n_packets=0, n_bytes=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    cookie=0x0, duration=20101.654s, table=0, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,1)
    cookie=0x0, duration=17215.496s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p1 actions=resubmit(,2)
    cookie=0x0, duration=19.570s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p2,vlan_tci=0x0000 actions=mod_vlan_vid:20,resubmit(,2)
    cookie=0x0, duration=19.568s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p3,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=19.568s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p4,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=18761.602s, table=1, n_packets=0, n_bytes=0, priority=0 actions=drop
   ```
5. 测试Table 1
   + 验证trunk port包
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=1,vlan_tci=5
   Flow: in_port=1,vlan_tci=0x0005,vlan_tci1=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=1, priority 99
       resubmit(,2)
    3. No match.
       drop

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop
   ```
   + 验证access port包
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=2
   Flow: in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=2,vlan_tci=0x0000, priority 99
       mod_vlan_vid:20
       resubmit(,2)
    3. No match.
       drop

   Final flow: in_port=2,dl_vlan=20,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000
   Megaflow: recirc_id=0,eth,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop

   # ovs-appctl ofproto/trace br0 in_port=2,vlan_tci=5
   Flow: in_port=2,vlan_tci=0x0005,vlan_tci1=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. priority 0
       drop

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=2,vlan_tci=0x0005,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop
   ```
## step 4 Implementing Table 2: MAC+VLAN learning for Ingress Port
This table allows the switch we’re implementing to learn that the packet’s source MAC is located on the packet’s ingress port in the packet’s VLAN.
1. add flow
   ```terminal
   ovs-ofctl add-flow br0 "table=2 actions=learn(table=10, NXM_OF_VLAN_TCI[0..11], NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[], load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)"
   ```
   > learn action is an Open vSwitch extension to OpenFlow， modifies a flow table based on the content of the flow currentily being processed. 
   > + table10: Modify flow table 10. This will be the MAC learning table.
   > + NXM_OF_VLAN_TCI[0..11]: Make the flow that we add to flow table 10 match the same VLAN ID that the packet we’re currently processing contains. This effectively scopes the MAC learning entry to a single VLAN, which is the ordinary behavior for a VLAN-aware switch.
   > + NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[]: Make the flow that we add to flow table 10 match, as Ethernet destination, the Ethernet source address of the packet we’re currently processing.
   > + load:MXN_OF_IN_PORT[]->NXM_NX_REG0[0..15]: Whereas the preceding parts specify fields for the new flow to match, this specifies an action for the flow to take when it matches. The action is for the flow to <u>load the **ingress port** number of the current packet into register 0</u> (a special field that is an Open vSwitch extension to OpenFlow).
2. 查看此时流表
   ```terminal
   # ovs-ofctl dump-flows br0
    cookie=0x0, duration=25596.118s, table=0, n_packets=0, n_bytes=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
    cookie=0x0, duration=25309.304s, table=0, n_packets=0, n_bytes=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    cookie=0x0, duration=25185.301s, table=0, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,1)
    cookie=0x0, duration=22299.143s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p1 actions=resubmit(,2)
    cookie=0x0, duration=5103.217s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p2,vlan_tci=0x0000 actions=mod_vlan_vid:20,resubmit(,2)
    cookie=0x0, duration=5103.215s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p3,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=5103.215s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p4,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=23845.249s, table=1, n_packets=0, n_bytes=0, priority=0 actions=drop
    cookie=0x0, duration=76.381s, table=2, n_packets=0, n_bytes=0, actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)
   ```
3. 测试Table 2
   + 测试1
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=1,vlan_tci=20,dl_src=50:00:00:00:00:01 -generate
   Flow: in_port=1,vlan_tci=0x0014,vlan_tci1=0x0000,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=1, priority 99
       resubmit(,2)
    3. priority 32768
       learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
        -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
       resubmit(,3)
    4. No match.
       drop

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,vlan_tci=0x0014/0x1fff,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop 

   # ovs-ofctl dump-flows br0 table=10
    cookie=0x0, duration=1191.241s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
   ```
   > -generate使ofproto/trace强制执行边际效应side effect
   + 测试2
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=2,dl_src=50:00:00:00:00:01 -generate
   Flow: in_port=2,vlan_tci=0x0000,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=2,vlan_tci=0x0000, priority 99
       mod_vlan_vid:20
       resubmit(,2)
    3. priority 32768
       learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
        -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 priority=32768 actions=load:0x2->NXM_NX_REG0[0..15]
       resubmit(,3)
    4. No match.
       drop

   Final flow: in_port=2,dl_vlan=20,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00,dl_type=0x0000
   Megaflow: recirc_id=0,eth,in_port=2,vlan_tci=0x0000,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: drop

   # ovs-ofctl dump-flows br0 table=10
    cookie=0x0, duration=1034.300s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x2->NXM_NX_REG0[0..15]
   ```
## step 5 Implementing Table 3: Look Up Destination Port
This table figures out what port we should send the packet to based on the destination MAC and VLAN. That is, if we’ve learned the location of the destination (from table 2 processing some previous packet with that destination as its source)
1. 添加MAC地址查找规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=3 priority=50 actions=resubmit(,10),resubmit(,4)"
   ```
   > 第一个动作是resubmit(,10)，在table 10中寻找已学习到的mac地址记录  
   > 第二个动作是resubmit(,4)，继续packet pipline
2. 添加广播包flood规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=3 priority=99 dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,4)"
   ```
   > 广播包flood规则优先级为99，高于普通包规则优先级50，利用了reg0初始值为0的trick
3. 查看此时流表
   ```terminal
   # ovs-ofctl dump-flows br0
    cookie=0x0, duration=29246.150s, table=0, n_packets=0, n_bytes=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
    cookie=0x0, duration=28959.336s, table=0, n_packets=0, n_bytes=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    cookie=0x0, duration=28835.333s, table=0, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,1)
    cookie=0x0, duration=25949.175s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p1 actions=resubmit(,2)
    cookie=0x0, duration=8753.249s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p2,vlan_tci=0x0000 actions=mod_vlan_vid:20,resubmit(,2)
    cookie=0x0, duration=8753.247s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p3,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=8753.247s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p4,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=27495.281s, table=1, n_packets=0, n_bytes=0, priority=0 actions=drop
    cookie=0x0, duration=3726.413s, table=2, n_packets=0, n_bytes=0, actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)
    cookie=0x0, duration=132.142s, table=3, n_packets=0, n_bytes=0, priority=99,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,4)
    cookie=0x0, duration=1133.037s, table=3, n_packets=0, n_bytes=0, priority=50 actions=resubmit(,10),resubmit(,4)
    cookie=0x0, duration=3542.556s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x2->NXM_NX_REG0[0..15]
   ```
4. 验证table3
   ```terminal
   # ovs-appctl ofproto/trace br0 in_port=1,dl_vlan=20,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01 -generate
   Flow: in_port=1,dl_vlan=20,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=1, priority 99
       resubmit(,2)
    3. priority 32768
       learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
        -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
       resubmit(,3)
    4. priority 50
       resubmit(,10)
       1.  No match.
               drop
       resubmit(,4)
    5. No match.
       drop

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=1,dl_vlan=20,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01,dl_type=0x0000
   Datapath actions: drop

   # ovs-ofctl dump-flows br0 table=10
    cookie=0x0, duration=5131.781s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x2->NXM_NX_REG0[0..15]
    cookie=0x0, duration=703.895s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]

   # ovs-appctl ofproto/trace br0 in_port=1,dl_vlan=20,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01 -generate
   Flow: in_port=1,dl_vlan=20,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01,dl_type=0x0000

   bridge("br0")
   -------------
    0. priority 0
       resubmit(,1)
    1. in_port=1, priority 99
       resubmit(,2)
    2. priority 32768
       learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
        -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=90:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
       resubmit(,3)
    3. priority 50
       resubmit(,10)
       10. vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01, priority 32768
               load:0x1->NXM_NX_REG0[0..15]
       resubmit(,4)
    4. No match.
       drop

   Final flow: reg0=0x1,in_port=1,dl_vlan=20,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01,dl_type=0x0000
   Megaflow: recirc_id=0,eth,in_port=1,dl_vlan=20,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01,dl_type=0x0000
   Datapath actions: drop

   # ovs-ofctl dump-flows br0 table=10
    cookie=0x0, duration=5408.466s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x2->NXM_NX_REG0[0..15]
    cookie=0x0, duration=980.580s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
    cookie=0x0, duration=208.629s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=90:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
   ```
## step 6 Implementing Table 4: Output Processing
At entry to stage 4, we know that register 0 contains either the desired output port or is zero if the packet should be flooded. We also know that the packet’s VLAN is in its 802.1Q header, even if the VLAN was implicit because the packet came in on an access port.
1. 添加trunk port规则
   ```terminal
   # ovs-ofctl add-flow br0 "table=4 reg0=1 actions=1"
   ```
2. 添加access port规则
   ```terminal
   # ovs-ofctl add-flows br0 - <<'EOF'
   table=4 reg0=2 actions=strip_vlan,2
   table=4 reg0=3 actions=strip_vlan,3
   table=4 reg0=4 actions=strip_vlan,4
   EOF
   ```
3. 添加组播与广播规则
   ```terminal
   # ovs-ofctl add-flow br0 - <<'EOF'
   table=4 reg0=0 priority=99 dl_vlan=20 actions=1,strip_vlan,2
   table=4 reg0=0 priority=99 dl_vlan=30 actions=1,strip_vlan,3,4
   table=4 reg0=0 priority=50 actions=1
   EOF
   ```
   > 对于组播/广播规则而言，保证两点：1、对于access port而言，只能广播到同一个VLAN ID中的端口；2、对于trunk port而言，接收所有广播包
4. 查看流表
   ```terminal
   # ovs-ofctl dump-flows br0
    cookie=0x0, duration=33901.604s, table=0, n_packets=0, n_bytes=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
    cookie=0x0, duration=33614.791s, table=0, n_packets=0, n_bytes=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0 actions=drop
    cookie=0x0, duration=33490.788s, table=0, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,1)
    cookie=0x0, duration=30604.630s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p1 actions=resubmit(,2)
    cookie=0x0, duration=13408.704s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p2,vlan_tci=0x0000 actions=mod_vlan_vid:20,resubmit(,2)
    cookie=0x0, duration=13408.702s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p3,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=13408.702s, table=1, n_packets=0, n_bytes=0, priority=99,in_port=p4,vlan_tci=0x0000 actions=mod_vlan_vid:30,resubmit(,2)
    cookie=0x0, duration=32150.736s, table=1, n_packets=0, n_bytes=0, priority=0 actions=drop
    cookie=0x0, duration=8381.868s, table=2, n_packets=0, n_bytes=0, actions=learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]),resubmit(,3)
    cookie=0x0, duration=4787.597s, table=3, n_packets=0, n_bytes=0, priority=99,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,4)
    cookie=0x0, duration=5788.492s, table=3, n_packets=0, n_bytes=0, priority=50 actions=resubmit(,10),resubmit(,4)
    cookie=0x0, duration=2022.195s, table=4, n_packets=0, n_bytes=0, reg0=0x1 actions=output:p1
    cookie=0x0, duration=1770.903s, table=4, n_packets=0, n_bytes=0, reg0=0x2 actions=strip_vlan,output:p2
    cookie=0x0, duration=1770.902s, table=4, n_packets=0, n_bytes=0, reg0=0x3 actions=strip_vlan,output:p3
    cookie=0x0, duration=1770.901s, table=4, n_packets=0, n_bytes=0, reg0=0x4 actions=strip_vlan,output:p4
    cookie=0x0, duration=1414.364s, table=4, n_packets=0, n_bytes=0, priority=50,reg0=0 actions=output:p1
    cookie=0x0, duration=1414.366s, table=4, n_packets=0, n_bytes=0, priority=99,reg0=0,dl_vlan=20 actions=output:p1,strip_vlan,output:p2
    cookie=0x0, duration=1414.364s, table=4, n_packets=0, n_bytes=0, priority=99,reg0=0,dl_vlan=30 actions=output:p1,strip_vlan,output:p3,output:p4
    cookie=0x0, duration=8198.011s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x2->NXM_NX_REG0[0..15]
    cookie=0x0, duration=3770.125s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
    cookie=0x0, duration=2998.174s, table=10, n_packets=0, n_bytes=0, vlan_tci=0x0014/0x0fff,dl_dst=90:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
   ```
> **注意：**
> 上述规则依赖open flow标准行为，即，对于包输出动作而言，该动作不会将包转发至该包输入的端口，也即若包从p1进入，则输出阶段无论如何不会将包转发至p1；上述规则中的actions=1即上述规则的描述。
1. 测试Table 4
   + 广播、组播以及未知目的端口包
    ```terminal
    # ovs-appctl pfproto/trace br0 in_port=1,dl_dst=ff:ff:ff:ff:ff:ff,dl_vlan=30
    Flow: in_port=1,dl_vlan=30,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=00:00:00:00:00:00,dl_dst=ff:ff:ff:ff:ff:ff,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=1, priority 99
       resubmit(,2)
    3. priority 32768
       learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
        >> suppressing side effects, so learn action ignored
       resubmit(,3)
    4. dl_dst=01:00:00:00:00:00/01:00:00:00:00:00, priority 99
       resubmit(,4)
    5. reg0=0,dl_vlan=30, priority 99
       output:1
        >> skipping output to input port
       strip_vlan
       output:3
       output:4

   Final flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=ff:ff:ff:ff:ff:ff,dl_type=0x0000
   Megaflow: recirc_id=0,eth,in_port=1,dl_vlan=30,dl_vlan_pcp=0,dl_src=00:00:00:00:00:00,dl_dst=ff:ff:ff:ff:ff:f0/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: pop_vlan,4,5
    ```
    > datapath actions : pop_vlan 4 5
    ```terminal
    # ovs-appctl ofproto/trace br0 in_port=3,dl_dst=ff:ff:ff:ff:ff:ff
    Flow: in_port=3,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=ff:ff:ff:ff:ff:ff,dl_type=0x0000

   bridge("br0")
   -------------
    1. priority 0
       resubmit(,1)
    2. in_port=3,vlan_tci=0x0000, priority 99
       mod_vlan_vid:30
       resubmit(,2)
    3. priority 32768
       learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
        >> suppressing side effects, so learn action ignored
       resubmit(,3)
    4. dl_dst=01:00:00:00:00:00/01:00:00:00:00:00, priority 99
       resubmit(,4)
    5. reg0=0,dl_vlan=30, priority 99
       output:1
       strip_vlan
       output:3
        >> skipping output to input port
       output:4

   Final flow: unchanged
   Megaflow: recirc_id=0,eth,in_port=3,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=ff:ff:ff:ff:ff:f0/ff:ff:ff:ff:ff:f0,dl_type=0x0000
   Datapath actions: push_vlan(vid=30,pcp=0),2,pop_vlan,5
    ```  
    + Mac learning  
      a. learn a MAC on port p1 in VLAN 30:
      ```terminal
        # ovs-appctl ofproto/trace br0 in_port=1,dl_vlan=30,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01 -generate
        Flow: in_port=1,dl_vlan=30,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01,dl_type=0x0000

       bridge("br0")
       -------------
        1. priority 0
           resubmit(,1)
        2. in_port=1, priority 99
           resubmit(,2)
        3. priority 32768
           learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
            -> table=10 vlan_tci=0x001e/0x0fff,dl_dst=10:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
           resubmit(,3)
        4. priority 50
           resubmit(,10)
           1.  No match.
                   drop
           resubmit(,4)
        5. reg0=0,dl_vlan=30, priority 99
           output:1
            >> skipping output to input port
           strip_vlan
           output:3
           output:4

       Final flow: in_port=1,vlan_tci=0x0000,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01,dl_type=0x0000
       Megaflow: recirc_id=0,eth,in_port=1,dl_vlan=30,dl_vlan_pcp=0,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01,dl_type=0x0000
       Datapath actions: pop_vlan,4,5
      ```  
     
       b. reverse the MACs and learn mac
       ```terminal
       # ovs-appctl ofproto/trace br0 in_port=4,dl_src=20:00:00:00:00:01,dl_dst=10:00:00:00:00:01 -generate
       Flow: in_port=4,vlan_tci=0x0000,dl_src=20:00:00:00:00:01,dl_dst=10:00:00:00:00:01,dl_type=0x0000

       bridge("br0")
       -------------
        0. priority 0
           resubmit(,1)
        1. in_port=4,vlan_tci=0x0000, priority 99
           mod_vlan_vid:30
           resubmit(,2)
        2. priority 32768
           learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
            -> table=10 vlan_tci=0x001e/0x0fff,dl_dst=20:00:00:00:00:01 priority=32768 actions=load:0x4->NXM_NX_REG0[0..15]
           resubmit(,3)
        3. priority 50
           resubmit(,10)
           10. vlan_tci=0x001e/0x0fff,dl_dst=10:00:00:00:00:01, priority 32768
                   load:0x1->NXM_NX_REG0[0..15]
           resubmit(,4)
        4. reg0=0x1, priority 32768
           output:1

       Final flow: reg0=0x1,in_port=4,dl_vlan=30,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=20:00:00:00:00:01,dl_dst=10:00:00:00:00:01,dl_type=0x0000
       Megaflow: recirc_id=0,eth,in_port=4,vlan_tci=0x0000,dl_src=20:00:00:00:00:01,dl_dst=10:00:00:00:00:01,dl_type=0x0000
       Datapath actions: push_vlan(vid=30,pcp=0),2

       ```
       c. 重新执行第一条指令
       ```terminal
       # ovs-appctl ofproto/trace br0 in_port=1,dl_vlan=30,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01 -generate
       Flow: in_port=1,dl_vlan=30,dl_vlan_pcp=0,vlan_tci1=0x0000,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01,dl_type=0x0000

       bridge("br0")
       -------------
        0. priority 0
           resubmit(,1)
        1. in_port=1, priority 99
           resubmit(,2)
        2. priority 32768
           learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
            -> table=10 vlan_tci=0x001e/0x0fff,dl_dst=10:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
           resubmit(,3)
        3. priority 50
           resubmit(,10)
           10. vlan_tci=0x001e/0x0fff,dl_dst=20:00:00:00:00:01, priority 32768
                   load:0x4->NXM_NX_REG0[0..15]
           resubmit(,4)
        4. reg0=0x4, priority 32768
           strip_vlan
           output:4

       Final flow: reg0=0x4,in_port=1,vlan_tci=0x0000,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01,dl_type=0x0000
       Megaflow: recirc_id=0,eth,in_port=1,dl_vlan=30,dl_vlan_pcp=0,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01,dl_type=0x0000
       Datapath actions: pop_vlan,5

       ```