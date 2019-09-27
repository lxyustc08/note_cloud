# Basic scenairo
## 目的
建立一个Open vSwitch flow tables,支持VLAN、MAC-learning，拥有4个端口：
+ p1:
  + 汇聚端口trunck port，对应OpenFlow中的端口1
+ p2
  + access port，VLAN ID为20， 对应OpenFlow中的端口2
+ p3,p4
  + 均为access port, VLAN ID为30，分别对应OpenFlow中的端口3和4

交换机建立5张流表，每张流表对应switch pipeline中的一个阶段
+ Table 0
  + Admission control
+ Table 1
  + VLAN input processing
+ Table 2
  + Learn source MAV and VLAN for ingress port
+ Table 3
  + Look up learned port for destination MAC and VLAN
+ Table 4
  + Output processing