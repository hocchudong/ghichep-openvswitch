# Open vSwitch với Mininet - part 2

*Bài lab này mục đích để làm quen một số đồ hình cơ bản khi làm việc với mininet. Chú ý rằng khi cài mininet thì open vswitch cũng được cài theo, switch mặc định minine hỗ trợ cũng là open vswitch.*

*Nguồn: [https://ervikrant06.wordpress.com/2015/09/17/learning-ovs-open-vswitch-using-mininet-part-2/](https://ervikrant06.wordpress.com/2015/09/17/learning-ovs-open-vswitch-using-mininet-part-2/)*

## Trường hợp 1: Tạo topo với 4 host và một switch

- Tạo topo:

```sh
root@cloud:~# mn --topo=single,4
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
mininet>
```

- Kiểm tra trạng thái của OVS bridge:

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
        Port "s1-eth2"
            Interface "s1-eth2"
        Port "s1"
            Interface "s1"
                type: internal
        Port "s1-eth3"
            Interface "s1-eth3"
        Port "s1-eth4"
            Interface "s1-eth4"
    ovs_version: "2.6.1"
```

- Kiểm tra số lượng nodes và thông tin liên quan các nodes:

```sh
mininet> nodes
available nodes are:
c0 h1 h2 h3 h4 s1
mininet> dump
<Host h1: h1-eth0:10.0.0.1 pid=24948>
<Host h2: h2-eth0:10.0.0.2 pid=24950>
<Host h3: h3-eth0:10.0.0.3 pid=24952>
<Host h4: h4-eth0:10.0.0.4 pid=24954>
<OVSSwitch s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None,s1-eth3:None,s1-eth4:None pid=24959>
<Controller c0: 127.0.0.1:6633 pid=24941>
mininet> net
h1 h1-eth0:s1-eth1
h2 h2-eth0:s1-eth2
h3 h3-eth0:s1-eth3
h4 h4-eth0:s1-eth4
s1 lo:  s1-eth1:h1-eth0 s1-eth2:h2-eth0 s1-eth3:h3-eth0 s1-eth4:h4-eth0
c0
```

## Trường hợp 2: Tạo topo tuyến tính với 4 nodes:

- Tạo topo:

```sh
root@cloud:~# mn --topo=linear,4
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 h4
*** Adding switches:
s1 s2 s3 s4
*** Adding links:
(h1, s1) (h2, s2) (h3, s3) (h4, s4) (s2, s1) (s3, s2) (s4, s3)
*** Configuring hosts
h1 h2 h3 h4
*** Starting controller
c0
*** Starting 4 switches
s1 s2 s3 s4 ...
*** Starting CLI:
```

- Kiểm tra các OVS bridge, ta sẽ thấy 4 ovs bridge được tạo ra:

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
    Bridge "s4"
        Controller "ptcp:6637"
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port "s4"
            Interface "s4"
                type: internal
        Port "s4-eth1"
            Interface "s4-eth1"
        Port "s4-eth2"
            Interface "s4-eth2"
    Bridge "s3"
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        Controller "ptcp:6636"
        fail_mode: secure
        Port "s3-eth2"
            Interface "s3-eth2"
        Port "s3"
            Interface "s3"
                type: internal
        Port "s3-eth1"
            Interface "s3-eth1"
        Port "s3-eth3"
            Interface "s3-eth3"
    Bridge "s2"
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        Controller "ptcp:6635"
        fail_mode: secure
        Port "s2"
            Interface "s2"
                type: internal
        Port "s2-eth3"
            Interface "s2-eth3"
        Port "s2-eth1"
            Interface "s2-eth1"
        Port "s2-eth2"
            Interface "s2-eth2"
    ovs_version: "2.6.1"
```

- Kiểm tra kết nối và các liên kết giữa các nodes:

```sh
mininet> nodes
available nodes are:
c0 h1 h2 h3 h4 s1 s2 s3 s4
mininet> links
h1-eth0<->s1-eth1 (OK OK)
h2-eth0<->s2-eth1 (OK OK)
h3-eth0<->s3-eth1 (OK OK)
h4-eth0<->s4-eth1 (OK OK)
s2-eth2<->s1-eth2 (OK OK)
s3-eth2<->s2-eth3 (OK OK)
s4-eth2<->s3-eth3 (OK OK)
```

- Topo tuyến tính sẽ có dạng như sau:

```sh
			h2	  h3
            |     |
h1----s1----s2----s3----s4----h4
	   \      \  /      /
	    \______c0______/
```

## Trường hợp 3: Tạo topo dạng cây với 4 host và 3 switch:
- Thực hiện các command tương tự như hai topo trên, ta sẽ có topo tree tương tự như sau:

```sh
             s1 
   h1___   /  | \	___h4
		\ /   |  \ /
   h2----s2   |   s3----h3
          \   |  /
		   \_c0_/	
```
