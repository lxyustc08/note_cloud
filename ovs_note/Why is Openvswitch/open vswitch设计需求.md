# Open vswitch 设计需求
## 状态的可流动性
1. All network state associated with a network entity (say a virtual machine) should be easily identifiable and migratable between different hosts.   
   > network state includes traditional “soft state” (such as an entry in an L2 learning table), L3 forwarding state, policy routing state, ACLs, QoS policy, monitoring configuration (e.g. NetFlow, IPFIX, sFlow), etc.
2. Open vSwitch has support for both configuring and migrating both slow (configuration) and fast network state between instances. 
   > For example, if a VM migrates between end-hosts, it is possible to not only migrate associated configuration (SPAN rules, ACLs, QoS) but any live network state (including, for example, existing state which may be difficult to reconstruct). Further, Open vSwitch state is typed and backed by a real data-model allowing for the development of structured automation systems.
## 网络动态变化的响应
1. simple accounting and visibility support such as NetFlow, IPFIX, and sFlow.
2. a network state database (OVSDB) that supports remote triggers.
3. supports OpenFlow as a method of exporting remote access to control traffic. 
## 逻辑标签的维护
1. Open vSwitch includes multiple methods for specifying and maintaining tagging rules, all of which are accessible to a remote process for orchestration.
2. in many cases these tagging rules are stored in an optimized form so they don’t have to be coupled with a heavyweight network device.
3. Open vSwitch supports a GRE implementation that can handle thousands of simultaneous GRE tunnels and supports remote configuration for tunnel creation, configuration, and tear-down. 
## 硬件整合能力
1. Open vSwitch’s forwarding path (the in-kernel datapath) is designed to be amenable to “offloading” packet processing to hardware chipsets, whether housed in a classic hardware switch chassis or in an end-host NIC. 
   >  This allows for the Open vSwitch control path to be able to both control a pure software implementation or a hardware switch.