## Hướng dẫn cơ bản về Mininet

---

*Mininet là phần mềm giả lập mạng thông dụng cho phép tạo switch, host và kết nối giữa chúng để phục vụ mục đích kiểm thử*

*Nguồn: [https://ervikrant06.wordpress.com/2015/09/17/learning-ovs-open-vswitch-using-mininet-part-1/](https://ervikrant06.wordpress.com/2015/09/17/learning-ovs-open-vswitch-using-mininet-part-1/)*

### Bước 1: Cài đặt Mininet theo hướng dẫn từ trang chủ. Chuyển sang người dùng quản trị. Tạo một đồ hình mặc định với command `mn`

```sh
root@cloud:~# mn
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2
*** Adding switches:
s1
*** Adding links:
(h1, s1) (h2, s1)
*** Configuring hosts
h1 h2
*** Starting controller
c0
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet>
```

### Bước 2: Mininet cho phép sử dụng một số command từ trình shell mặc định của linux. Để thực hiện điều đó, thêm `sh` vào trước mỗi lệnh trên trình shell của mininet:

```sh
mininet> sh ifconfig -s
Iface   MTU Met   RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0       1500 0        23      0      0 0            65      0      0      0 BMRU
eth1       1500 0       799      0      0 0           882      0      0      0 BMRU
lo        65536 0       129      0      0 0           129      0      0      0 LRU
s1-eth1    1500 0         8      0      0 0            15      0      0      0 BMRU
s1-eth2    1500 0         8      0      0 0            15      0      0      0 BMRU
```

### Bước 3: Kiểm tra cấu hình các OVS bridge sau khi có được topo mặc định (hai hosts và một switch):

```sh
mininet> sh ovs-vsctl show
6e0731e1-3e2b-4f28-9f32-f35994700db9
    Bridge "s1"
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        Controller "ptcp:6634"
        fail_mode: secure
        Port "s1-eth1"
            Interface "s1-eth1"
        Port "s1"
            Interface "s1"
                type: internal
        Port "s1-eth2"
            Interface "s1-eth2"
    ovs_version: "2.6.1"
```

### Bước 4: Kiểm tra đồ hình mặc định:
- Kiểm tra các nodes, ở đây topo mặc định sẽ bao gồm hai host __h1__, __h2__, một switch __s1__ kết nối hai host và kết nối với controller __c0__.

```sh
mininet> nodes
available nodes are:
c0 h1 h2 s1
```

- Kiểm tra thông tin chi tiết hơn về mỗi thành phần trong đồ hình sử dụng lệnh `dump`:

```sh
mininet> dump
<Host h1: h1-eth0:10.0.0.1 pid=5969>
<Host h2: h2-eth0:10.0.0.2 pid=5971>
<OVSSwitch s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None pid=5976>
<Controller c0: 127.0.0.1:6633 pid=5962>
```

- Kiểm tra kết nối giữa các node sử dụng lệnh `net`:

```sh
mininet> net
h1 h1-eth0:s1-eth1
h2 h2-eth0:s1-eth2
s1 lo:  s1-eth1:h1-eth0 s1-eth2:h2-eth0
c0
```

- Ping thử giữa hai host:

```sh
mininet> h1 ping -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=29.3 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.490 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.079 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2022ms
rtt min/avg/max/mdev = 0.079/9.969/29.339/13.697 ms
```

- Thử vài command đơn giản trên host bất kì:

```sh
mininet> h1 ifconfig
h1-eth0   Link encap:Ethernet  HWaddr 4e:81:f1:24:1b:5a
          inet addr:10.0.0.1  Bcast:10.255.255.255  Mask:255.0.0.0
          inet6 addr: fe80::4c81:f1ff:fe24:1b5a/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:20 errors:0 dropped:0 overruns:0 frame:0
          TX packets:13 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1584 (1.5 KB)  TX bytes:1026 (1.0 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:10 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:584 (584.0 B)  TX bytes:584 (584.0 B)
```

- Kiểm tra kết nối giữa toàn bộ các host sử dụng command `pingall`:

```sh
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2
h2 -> h1
*** Results: 0% dropped (2/2 received)
```

- Thoát khỏi mininet:

```sh
mininet> exit
*** Stopping 1 controllers
c0
*** Stopping 4 terms
*** Stopping 2 links
..
*** Stopping 1 switches
s1
*** Stopping 2 hosts
h1 h2
*** Done
completed in 744.784 seconds
```

- Xóa toàn bộ các zombie process liên quan tới các thao tác với mininet ở trên:

```sh
root@cloud:~# mn -c
*** Removing excess controllers/ofprotocols/ofdatapaths/pings/noxes
killall controller ofprotocol ofdatapath ping nox_core lt-nox_core ovs-openflowd ovs-controller udpbwtest mnexec ivs 2> /dev/null
killall -9 controller ofprotocol ofdatapath ping nox_core lt-nox_core ovs-openflowd ovs-controller udpbwtest mnexec ivs 2> /dev/null
pkill -9 -f "sudo mnexec"
*** Removing junk from /tmp
rm -f /tmp/vconn* /tmp/vlogs* /tmp/*.out /tmp/*.log
*** Removing old X11 tunnels
*** Removing excess kernel datapaths
ps ax | egrep -o 'dp[0-9]+' | sed 's/dp/nl:/'
***  Removing OVS datapaths
ovs-vsctl --timeout=1 list-br
ovs-vsctl --timeout=1 list-br
*** Removing all links of the pattern foo-ethX
ip link show | egrep -o '([-_.[:alnum:]]+-eth[[:digit:]]+)'
ip link show
*** Killing stale mininet node processes
pkill -9 -f mininet:
*** Shutting down stale tunnels
pkill -9 -f Tunnel=Ethernet
pkill -9 -f .ssh/mn
rm -f ~/.ssh/mn/*
*** Cleanup complete.
```
