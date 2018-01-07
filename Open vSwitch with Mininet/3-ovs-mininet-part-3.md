# Open vSwitch với Mininet - part 3

*Bài lab cơ bản về chủ đề OpenFlow: kiểm tra flow table entries thêm vào bởi SDN controller trên switch*

*Nguồn: [https://ervikrant06.wordpress.com/2015/09/18/learning-ovs-open-vswitch-using-mininet-part-3/](https://ervikrant06.wordpress.com/2015/09/18/learning-ovs-open-vswitch-using-mininet-part-3/)*

### Bước 1: Tạo topo gồm 4 host nối vào một switch sử dụng tùy chọn __--mac__ để giữ địa chỉ MAC cho các host:

```sh
root@cloud:~# mn --topo=single,4 --mac
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
c0
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> dump
<Host h1: h1-eth0:10.0.0.1 pid=64818>
<Host h2: h2-eth0:10.0.0.2 pid=64828>
<Host h3: h3-eth0:10.0.0.3 pid=64830>
<Host h4: h4-eth0:10.0.0.4 pid=64832>
<OVSSwitch s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None,s1-eth3:None,s1-eth4:None pid=64837>
<Controller c0: 127.0.0.1:6633 pid=64803>
```

### Bước 2: Thực hiện dump cấu hình của các port trên OVS bridge. Các host kết nối với các port riêng biệt trên switch. Command dưới dây cho phép match số hiệu port với tên port. Trong các flow table, ta có thể chỉ thấy được số hiệu port, do đó command này sẽ hữu ích để tiện theo dõi hơn:

```sh
mininet> sh ovs-ofctl dump-ports-desc s1
OFPST_PORT_DESC reply (xid=0x2):
 1(s1-eth1): addr:6a:8b:d3:b6:0e:3e
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(s1-eth2): addr:c6:5c:48:b3:e7:59
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 3(s1-eth3): addr:1e:88:de:37:63:36
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 4(s1-eth4): addr:ca:48:88:4d:34:b8
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(s1): addr:a2:92:32:13:53:41
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
```

### Bước 3: Kiểm tra thông tin thống kê của mỗi port sử dụng command sau:

```sh
mininet> sh ovs-ofctl dump-ports s1
OFPST_PORT reply (xid=0x2): 5 ports
  port LOCAL: rx pkts=0, bytes=0, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=0, bytes=0, drop=28, errs=0, coll=0
  port  4: rx pkts=9, bytes=738, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=29, bytes=2322, drop=0, errs=0, coll=0
  port  1: rx pkts=9, bytes=738, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=29, bytes=2322, drop=0, errs=0, coll=0
  port  2: rx pkts=9, bytes=738, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=29, bytes=2322, drop=0, errs=0, coll=0
  port  3: rx pkts=9, bytes=738, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=29, bytes=2322, drop=0, errs=0, coll=0
```

### Bước 4: In ra flow tables do __ovs-vswitchd__ quản lý. Ta sẽ thấy chưa có entry nào do chưa có bất kì thao tác kết nối nào giữa các host:

```sh
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
```

Thử kiểm tra controller điều khiển switch s1:

```sh
mininet> sh ovs-vsctl get-controller s1
ptcp:6634
tcp:127.0.0.1:6633
```

### Bước 5: Tạo traffic đơn giản bằng cách ping giữa host h1 và host h2. Ta sẽ thấy rằng gói tin đầu tiên có Round Trip Time lâu hơn do nó là gói tin đầu tiên của flow mới từ host 1 gửi tới, switch chưa xử lý được sẽ gửi lên controller xử lý rồi mới gửi lại switch để chuyển tiếp sang host 2. Những gói tin tiếp theo do cùng flow nên thực hiện flow matching thành công ngay tại switch và chuyển tiếp trực tiếp sang host 2 mà không cần hỏi lên controller, nên Round Trip Time sẽ ngắn hơn.

```sh
mininet> h1 ping -c5 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=33.5 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.504 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.079 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.075 ms
64 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=0.081 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 0.075/6.851/33.518/13.334 ms
```

### Bước 6: Dump flow tables một lần nữa. Ta sẽ thây ARP rule được thêm vào flow table. Chú ý output là các port mà gói tin sẽ được forward ở đầu ra của pipeline trên switch. Số hiệu port tương ứng với các port dump ở bước 2:

```sh
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=5.966s, table=0, n_packets=0, n_bytes=0, idle_timeout=60, idle_age=5, priority=65535,arp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,arp_spa=10.0.0.2,arp_tpa=10.0.0.1,arp_op=2 actions=output:1
 cookie=0x0, duration=0.937s, table=0, n_packets=0, n_bytes=0, idle_timeout=60, idle_age=0, priority=65535,arp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,arp_spa=10.0.0.2,arp_tpa=10.0.0.1,arp_op=1 actions=output:1
 cookie=0x0, duration=0.928s, table=0, n_packets=0, n_bytes=0, idle_timeout=60, idle_age=0, priority=65535,arp,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,arp_spa=10.0.0.1,arp_tpa=10.0.0.2,arp_op=2 actions=output:2
 cookie=0x0, duration=5.957s, table=0, n_packets=4, n_bytes=392, idle_timeout=60, idle_age=1, priority=65535,icmp,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,nw_src=10.0.0.1,nw_dst=10.0.0.2,nw_tos=0,icmp_type=8,icmp_code=0 actions=output:2
 cookie=0x0, duration=5.953s, table=0, n_packets=4, n_bytes=392, idle_timeout=60, idle_age=1, priority=65535,icmp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,nw_src=10.0.0.2,nw_dst=10.0.0.1,nw_tos=0,icmp_type=0,icmp_code=0 actions=output:1
```