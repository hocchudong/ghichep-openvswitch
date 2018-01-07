# Open vSwitch với Mininet - part 4

*Bài lab này tiếp nối chủ đề OpenFlow của part 3*

*Nguồn: [https://ervikrant06.wordpress.com/2015/09/18/learning-ovs-open-vswitch-using-mininet-part-4/](https://ervikrant06.wordpress.com/2015/09/18/learning-ovs-open-vswitch-using-mininet-part-4/)*

### Bước 1: Xóa các flow entry hiện tại trên flow table của switch:

```sh
mininet> sh ovs-ofctl del-flows s1
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
```

### Bước 2: Sử dụng tcpdump ở chế độ background để bắt gói tin ping từ h1 sang h2:

- Khởi động __tcpdump__ ở chế độ background trên trình shell chính của linux, lắng nghe trên cổng loopback:

```sh
root@cloud:~# tcpdump -s0 -i lo -w /tmp/h1pingh2.pcap &
[1] 103253
root@cloud:~# tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
```

- Ping từ h1 sang h2 trên shell của mininet: 

```sh
mininet> h1 ping -c5 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=39.2 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=9.50 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.344 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.080 ms
64 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=0.071 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 0.071/9.842/39.214/15.125 ms
```

- Kill tiến trình tcpdump: `kill -9 103253`

### Bước 3: Dump flow tables trên switch:

```sh
mininet>  sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=6.666s, table=0, n_packets=0, n_bytes=0, idle_timeout=60, idle_age=6, priority=65535,arp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,arp_spa=10.0.0.2,arp_tpa=10.0.0.1,arp_op=2 actions=output:1
 cookie=0x0, duration=1.639s, table=0, n_packets=0, n_bytes=0, idle_timeout=60, idle_age=1, priority=65535,arp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,arp_spa=10.0.0.2,arp_tpa=10.0.0.1,arp_op=1 actions=output:1
 cookie=0x0, duration=1.630s, table=0, n_packets=0, n_bytes=0, idle_timeout=60, idle_age=1, priority=65535,arp,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,arp_spa=10.0.0.1,arp_tpa=10.0.0.2,arp_op=2 actions=output:2
 cookie=0x0, duration=6.657s, table=0, n_packets=4, n_bytes=392, idle_timeout=60, idle_age=2, priority=65535,icmp,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,nw_src=10.0.0.1,nw_dst=10.0.0.2,nw_tos=0,icmp_type=8,icmp_code=0 actions=output:2
 cookie=0x0, duration=6.653s, table=0, n_packets=4, n_bytes=392, idle_timeout=60, idle_age=2, priority=65535,icmp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,nw_src=10.0.0.2,nw_dst=10.0.0.1,nw_tos=0,icmp_type=0,icmp_code=0 actions=output:1
 ```

### Bước 4: Mở file tcpdump đã ghi ra và lọc ra các bản tin OpenFlow: 

```sh
root@cloud:~# tshark -tad -n -r /tmp/h1pingh2.pcap -Y of
 1 2017-06-22 00:28:45.855151 00:00:00:00:00:01 -> ff:ff:ff:ff:ff:ff OpenFlow 126 Type: OFPT_PACKET_IN
 2 2017-06-22 00:28:45.863446 00:00:00:00:00:01 -> ff:ff:ff:ff:ff:ff OpenFlow 132 Type: OFPT_PACKET_OUT
 4 2017-06-22 00:28:45.864283 00:00:00:00:00:02 -> 00:00:00:00:00:01 OpenFlow 126 Type: OFPT_PACKET_IN
 5 2017-06-22 00:28:45.865362    127.0.0.1 -> 127.0.0.1    OpenFlow 146 Type: OFPT_FLOW_MOD
 6 2017-06-22 00:28:45.873067 00:00:00:00:00:02 -> 00:00:00:00:00:01 OpenFlow 132 Type: OFPT_PACKET_OUT
 8 2017-06-22 00:28:45.873680     10.0.0.1 -> 10.0.0.2     OpenFlow 182 Type: OFPT_PACKET_IN
 9 2017-06-22 00:28:45.873940    127.0.0.1 -> 127.0.0.1    OpenFlow 146 Type: OFPT_FLOW_MOD
10 2017-06-22 00:28:45.877558     10.0.0.1 -> 10.0.0.2     OpenFlow 188 Type: OFPT_PACKET_OUT
12 2017-06-22 00:28:45.878057     10.0.0.2 -> 10.0.0.1     OpenFlow 182 Type: OFPT_PACKET_IN
13 2017-06-22 00:28:45.878475    127.0.0.1 -> 127.0.0.1    OpenFlow 146 Type: OFPT_FLOW_MOD
14 2017-06-22 00:28:45.882282     10.0.0.2 -> 10.0.0.1     OpenFlow 188 Type: OFPT_PACKET_OUT
16 2017-06-22 00:28:50.568008    127.0.0.1 -> 127.0.0.1    OpenFlow 74 Type: OFPT_ECHO_REQUEST
17 2017-06-22 00:28:50.568617    127.0.0.1 -> 127.0.0.1    OpenFlow 74 Type: OFPT_ECHO_REPLY
19 2017-06-22 00:28:50.888683 00:00:00:00:00:02 -> 00:00:00:00:00:01 OpenFlow 126 Type: OFPT_PACKET_IN
20 2017-06-22 00:28:50.891608    127.0.0.1 -> 127.0.0.1    OpenFlow 146 Type: OFPT_FLOW_MOD
22 2017-06-22 00:28:50.899306 00:00:00:00:00:02 -> 00:00:00:00:00:01 OpenFlow 132 Type: OFPT_PACKET_OUT
24 2017-06-22 00:28:50.900205 00:00:00:00:00:01 -> 00:00:00:00:00:02 OpenFlow 126 Type: OFPT_PACKET_IN
25 2017-06-22 00:28:50.900602    127.0.0.1 -> 127.0.0.1    OpenFlow 146 Type: OFPT_FLOW_MOD
26 2017-06-22 00:28:50.907594 00:00:00:00:00:01 -> 00:00:00:00:00:02 OpenFlow 132 Type: OFPT_PACKET_OUT
28 2017-06-22 00:28:55.568253    127.0.0.1 -> 127.0.0.1    OpenFlow 74 Type: OFPT_ECHO_REQUEST
29 2017-06-22 00:28:55.569046    127.0.0.1 -> 127.0.0.1    OpenFlow 74 Type: OFPT_ECHO_REPLY
31 2017-06-22 00:29:00.568402    127.0.0.1 -> 127.0.0.1    OpenFlow 74 Type: OFPT_ECHO_REQUEST
32 2017-06-22 00:29:00.569312    127.0.0.1 -> 127.0.0.1    OpenFlow 74 Type: OFPT_ECHO_REPLY
```

Từ kết quả trên có thể thấy các frame 5, 9, 13, 20, 25 là các bản tin tương ứng hành động add flow từ controller tạo ra 5 flow entries trên table 0 như kết quả dump flow table ở bước trước.

Thử mở frame bất kì để quan sát một số thông tin liên quan tới OpenFlow:

```sh
root@cloud:~# tshark -tad -n -r /tmp/h1pingh2.pcap -O of -Y 'frame.number  == 20'
...
Frame 20: 146 bytes on wire (1168 bits), 146 bytes captured (1168 bits)
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 6633 (6633), Dst Port: 50833 (50833), Seq: 625, Ack: 421, Len: 80
OpenFlow 1.0
    .000 0001 = Version: 1.0 (0x01)
    Type: OFPT_FLOW_MOD (14)
    Length: 80
    Transaction ID: 0
    Wildcards: 0
    In port: 2
    Ethernet source address: 00:00:00_00:00:02 (00:00:00:00:00:02)
    Ethernet destination address: 00:00:00_00:00:01 (00:00:00:00:00:01)
    Input VLAN id: 65535
    Input VLAN priority: 0
    Pad: 00
    Data not dissected yet
    Cookie: 0x0000000000000000
    Command: New flow (0)
    Idle time-out: 60
    hard time-out: 0
    Priority: 0
    Buffer Id: 0xffffffff
    Out port: 0
    Flags: 0
```

### Bước 5: Đợi sau hơn 1 phút, ping lại từ h1 sang h2:

```sh
mininet> h1 ping h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=9.05 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=9.67 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.351 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.071 ms
64 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=0.087 ms
64 bytes from 10.0.0.2: icmp_seq=6 ttl=64 time=0.083 ms
64 bytes from 10.0.0.2: icmp_seq=7 ttl=64 time=0.083 ms
64 bytes from 10.0.0.2: icmp_seq=8 ttl=64 time=0.059 ms
64 bytes from 10.0.0.2: icmp_seq=9 ttl=64 time=0.084 ms
^C
--- 10.0.0.2 ping statistics ---
9 packets transmitted, 9 received, 0% packet loss, time 8001ms
rtt min/avg/max/mdev = 0.059/2.171/9.674/3.847 ms
```

Ta sẽ thấy rằng giá trị `Idle time-out` (60s) thu được ở bước trên đã khiến cho flow inactive. Giá trị này cho biết sau bao lâu kể từ khi bắt được gói tin của flow thì flow sẽ inactive. Do đó, khi ta ping lại giữa hai host sau khoảng thời gian đó thì flow phải thiết lập lại. Do đó gói tin ban đầu lại có Round Trip Time lớn hơn các gói sau như giải thích ở part 3.

