# Open vSwitch với Mininet - part 6

*Mục tiêu bài lab là add flow vào table 0 và chuyển hướng lưu lượng mạng sang table 1 cùng với các action phù hợp.*

*Nguồn: [https://ervikrant06.wordpress.com/2015/09/19/learning-ovs-open-vswitch-using-mininet-part-6/](https://ervikrant06.wordpress.com/2015/09/19/learning-ovs-open-vswitch-using-mininet-part-6/)*

### Bước 1: Tạo topo không có network controller

```sh
root@cloud:~# mn --topo=single,4 --mac --controller=none
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 h4
*** Adding switches:
s1
*** Adding links:
(h1, s1) (h2, s1) (h3, s1) (h4, s1)
*** Configuring hosts
h1 h2 h3 h4
*** Starting controller

*** Starting 1 switches
s1 ...
*** Starting CLI:
```

### Bước 2: Thêm flow và kiểm tra hoạt động sử dụng ovs-appctl command:

```sh
mininet> sh ovs-ofctl add-flow s1 priority=1000,in_port=1,actions=output:2
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=10.639s, table=0, n_packets=0, n_bytes=0, idle_age=10, priority=1000,in_port=1 actions=output:2
```

Trace flows:

```sh
mininet> sh ovs-appctl ofproto/trace s1 in_port=1
Bridge: s1
Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

Rule: table=0 cookie=0 priority=1000,in_port=1
OpenFlow actions=output:2

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_type=0x0000
Datapath actions: 1
mininet> sh ovs-appctl ofproto/trace s1 in_port=1
Bridge: s1
Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

Rule: table=0 cookie=0 priority=1000,in_port=1
OpenFlow actions=output:2

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_type=0x0000
Datapath actions: 1
```

### Bước 3: Xóa flow thêm ở bước trước, đồng thời thêm flow cho phép chuyển hướng lưu lượng sang table 1 với từ khóa "resubmit(,1)":

```sh
mininet> sh ovs-ofctl del-flows s1
mininet> sh ovs-ofctl add-flow s1 "table=0,priority=1000,in_port=1,actions=resubmit(,1)"
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=9.453s, table=0, n_packets=0, n_bytes=0, idle_age=9, priority=1000,in_port=1 actions=resubmit(,1)
```

### Bước 4: Thêm flow vào bảng 1 cho phép chuyển tiếp lưu lượng ra port 2:

```sh
mininet> sh ovs-ofctl add-flow s1 "table=1,priority=1000,actions=output:2"
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=304.388s, table=0, n_packets=0, n_bytes=0, idle_age=304, priority=1000,in_port=1 actions=resubmit(,1)
 cookie=0x0, duration=8.001s, table=1, n_packets=0, n_bytes=0, idle_age=8, priority=1000 actions=output:2
```

Trace flows sử dụng __ovs-appctl__ để xác nhận gói tin đi qua pipeline của switch s1 phù hợp với ý định thêm flow ở các bước trên:

```sh
mininet> sh ovs-appctl ofproto/trace s1 in_port=1
Bridge: s1
Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

Rule: table=0 cookie=0 priority=1000,in_port=1
OpenFlow actions=resubmit(,1)

        Resubmitted flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000
        Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0 reg8=0x0 reg9=0x0 reg10=0x0 reg11=0x0 reg12=0x0 reg13=0x0 reg14=0x0 reg15=0x0
        Resubmitted  odp: drop
        Resubmitted megaflow: recirc_id=0,in_port=1,dl_type=0x0000
        Rule: table=1 cookie=0 priority=1000
        OpenFlow actions=output:2

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_type=0x0000
Datapath actions: 1
```