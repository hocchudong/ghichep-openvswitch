# Open vSwitch với Mininet - part 5

*Mục tiêu: Thay vì sử dụng controller, thực hiện add flow thủ công*

*Nguồn: [https://ervikrant06.wordpress.com/2015/09/19/learning-ovs-open-vswitch-using-mininet-part-5/](https://ervikrant06.wordpress.com/2015/09/19/learning-ovs-open-vswitch-using-mininet-part-5/)*

### Bước 1: Tạo topology không sử dụng controller

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
mininet> nodes
available nodes are:
h1 h2 h3 h4 s1
mininet> dump
<Host h1: h1-eth0:10.0.0.1 pid=26816>
<Host h2: h2-eth0:10.0.0.2 pid=26818>
<Host h3: h3-eth0:10.0.0.3 pid=26820>
<Host h4: h4-eth0:10.0.0.4 pid=26822>
<OVSSwitch s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None,s1-eth3:None,s1-eth4:None pid=26827>
```

### Bước 2: Thử ping giữa các host thấy các host không thể truyền thông với nhau do bridge không phải learning bridge trong khi đó không sử dụng controller để kiểm soát flow:

```sh
mininet> h1 ping -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
From 10.0.0.1 icmp_seq=2 Destination Host Unreachable
From 10.0.0.1 icmp_seq=3 Destination Host Unreachable

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2009ms
pipe 3
```

### Bước 3: Xác nhận tương quan giữa số hiệu port và tên port:

```sh
mininet> sh ovs-ofctl dump-ports-desc s1
OFPST_PORT_DESC reply (xid=0x2):
 1(s1-eth1): addr:02:22:46:ee:05:cf
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 2(s1-eth2): addr:4a:fb:3f:b6:b3:7e
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 3(s1-eth3): addr:be:48:21:49:ae:11
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 4(s1-eth4): addr:fe:ad:e2:ab:5a:89
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(s1): addr:ea:5f:6d:e0:06:4b
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
```

### Bước 4: Thêm flow thủ công cho switch s1 để s1 trở thành switch L2 bình thường:

```sh
mininet> sh ovs-ofctl add-flow s1 action=normal
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=15.321s, table=0, n_packets=0, n_bytes=0, idle_age=15, actions=NORMAL
```

Giờ đây các hosts có thể truyền thông với nhau bình thường:

```sh
mininet> h1 ping -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.636 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.072 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.070 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.070/0.259/0.636/0.266 ms
mininet> h3 ping -c3 h4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=0.586 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=0.050 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=64 time=0.080 ms

--- 10.0.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.050/0.238/0.586/0.246 ms
```

### Bước 5: Xóa flow ở bước 4 đi và thực hiện thêm flow thủ công hơn bằng việc thiết lập forwarding rule cho từng port:

- Xóa flow cũ:

```sh
mininet> sh ovs-ofctl del-flows s1
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
mininet> h1 ping -c2 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

- Thêm hai flows chỉ cho phép truyền thông giữa hai host h1 và h2:

```sh
mininet> sh ovs-ofctl add-flow s1 priority=1000,in_port=1,actions=output:2
mininet> sh ovs-ofctl add-flow s1 priority=1000,in_port=2,actions=output:1
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=17.461s, table=0, n_packets=0, n_bytes=0, idle_age=17, priority=1000,in_port=1 actions=output:2
 cookie=0x0, duration=11.725s, table=0, n_packets=0, n_bytes=0, idle_age=11, priority=1000,in_port=2 actions=output:1
```

Ping thử giữa các host để kiểm tra:

```sh
mininet> h1 ping -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.365 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.070 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.065 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.065/0.166/0.365/0.141 ms
mininet> h1 ping -c2 h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
From 10.0.0.1 icmp_seq=2 Destination Host Unreachable

--- 10.0.0.3 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1007ms
pipe 2
```

Có thể thấy rằng chỉ có hai host h1 và h2 truyền thông được với nhau, các host khác không thể truyền thông với nhau cũng như với hai host h1 và h2.

### Bước 6: Thêm flow mới với __action__ drop và độ ưu tiên cao hơn hai flow entry vừa thêm ở trên:

```sh
mininet> sh ovs-ofctl add-flow s1 priority=1001,actions=drop
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=10.486s, table=0, n_packets=0, n_bytes=0, idle_age=10, priority=1001 actions=drop
 cookie=0x0, duration=419.227s, table=0, n_packets=8, n_bytes=504, idle_age=380, priority=1000,in_port=1 actions=output:2
 cookie=0x0, duration=413.491s, table=0, n_packets=5, n_bytes=378, idle_age=392, priority=1000,in_port=2 actions=output:1
```

Thực hiện ping giữa hai host h1 và h2, kết quả là thất bại do flow entry vừa mới thêm có độ ưu tiên cao nhất nên được áp dụng, ngăn cản truyền thông giữa hai host:

```sh
mininet> h1 ping -c2 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1000ms
```

### Bước 7: Hủy bỏ flow entry "drop" và thử lại, hai host h1 và h2 lại truyền thông với nhau bình thường:

- Xóa flow entry "drop":

```sh
mininet> sh ovs-ofctl del-flows s1 --strict priority=1001
mininet> sh ovs-ofctl dump-flows
ovs-ofctl: 'dump-flows' command requires at least 1 arguments
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=807.631s, table=0, n_packets=8, n_bytes=504, idle_age=768, priority=1000,in_port=1 actions=output:2
 cookie=0x0, duration=801.896s, table=0, n_packets=5, n_bytes=378, idle_age=780, priority=1000,in_port=2 actions=output:1
```

- Ping lại giữa hai host h1 và h2, kết quả truyền thông bình thường:

```sh
mininet> h1 ping -c2 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.888 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.079 ms

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.079/0.483/0.888/0.405 ms
```

