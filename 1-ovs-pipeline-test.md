# Open vSwitch tutorial - pipeline testing
# Mục lục
## [1. Khởi động sandbox](#sandbox)
## [2. Kịch bản](#scenario)
## [3. Cài đặt](#install)
## [4. Triển khai Table 0: Admission Control](#tbl0)
## [5. Triển khai Table 1: VLAN input processing](#tbl1)
## [6. Triển khai Table 2: MAC+VLAN Learning for Ingress Port](#tbl2)
## [7. Triển khai Table 3: Look up destination port](#tbl3)
## [8. Triển khai Table 4: Output Processing](#tbl4)

---
*Nguồn: [http://docs.openvswitch.org/en/latest/tutorials/ovs-advanced](http://docs.openvswitch.org/en/latest/tutorials/ovs-advanced)*


## <a name="sandbox"></a> 1. Khởi động sandbox
- Cài đặt Open vSwitch từ mã nguồn (script cài đặt [tại đây](https://github.com/thaihust/ovs_nsh_patches/blob/master/start-ovs-deb-2.6.1.sh)). Môi trường cài đặt là ubuntu 14.04 hoặc 16.04.
- Chuyển vào thư mục tutorial của Open vSwitch: `cd ~/ovs/tutorial`
- Thực thi script `ovs-sandbox`: `./ovs-sandbox`
- Script trên sẽ thực hiện các thao tác sau:
	- Thư mục `sandbox` do phiên làm việc cũ sẽ bị xóa, đồng thời thư mục `sandbox` mới được tạo ra.
	- Cài đặt biến môi trường đặc biệt đảm bảo Open vSwitch sẽ nhìn vào thư mục sandbox và làm việc thay vì thư mục cài đặt Open vSwitch
	- Tạo cơ sở dữ liệu cấu hình trong thư mục sandbox
	- Khởi động `ovsdb-server` trong thư mục `sandbox`
	- Khởi động `ovs-vswitchd` trong thư mục `sandbox`
	- Khởi động trình shell trong thư mục `sandbox`
- Dưới góc nhìn của OVS thì các bridge tạo ra trên môi trường sandbox tương tự như bridge thường, nhưng network stack của hệ điều hành chủ không thể nhìn thấy được các bridge này nên không thể sử dụng các lệnh thông thường như `ip` hay `tcpdump`

## <a name="scenario"></a> 2. Kịch bản 
- Tutorial này tạo nên các Open vSwitch flow tables để phục vụ các tính năng VLAN, MAC learning của switch với 4 port:
	- p1: trunk port cho phép gói tin từ mọi VLAN, tương ứng với Open Flow port 1
	- p2: access port cho VLAN 20, tương ứng Open Flow port 20
	- p3, p4: cả hai port này đều phục vụ VLAN 30, tương ứng với Open Flow port 3 và port 4
	
- Tạo switch bao gồm 4 bảng chính, mỗi bảng sẽ triển khai một stage trong pipeline của switch:
	- Table 0: Admission control - cho phép kiểm soát các gói tin đầu vào ở mức cơ bản
	- Table 1: xử lý VLAN đầu vào
	- Table 2: học MAC và VLAN đối với ingress port
	- Table 3: tìm kiếm port đã học nhằm xác định port đầu ra của gói tin
	- Table 4: xử lý đầu ra
	
## <a name="install"></a> 3. Cài đặt
- Tạo bridge `br0` ở `fail-secure` mode để Open Flow table rỗng khi khởi tạo, nếu không Open Flow table sẽ khởi tạo một flow thực thi `normal` action.

```sh
ovs-vsctl add-br br0 -- set Bridge br0 fail-mode=secure
```

- Tạo các port p1, p2, p3, p4 với tùy chọn `ofport_request` để đảm bảo port p1 gán cho Open Flow port 1, p2 được gán cho Open Flow port 2 và tương tự như vậy với hai port còn lại:

```sh
for i in 1 2 3 4; do
    ovs-vsctl add-port br0 p$i -- set Interface p$i ofport_request=$i
    ovs-ofctl mod-port br0 p$i up
done
```

## <a name="tbl0"></a> 4. Triển khai Table 0: Admission Control
- Table 0 là bảng đầu tiên gói tin đi qua đầu tiên, được sử dụng để bỏ qua các gói tin vì một lý do nào đó hoặc gói tin không hợp lệ. Trong trường hợp này, các gói tin với địa chỉ nguồn multicast được goi là không hợp lệ và do đó ta thêm flow để hủy chúng:

```sh
ovs-ofctl add-flow br0 \
    "table=0, dl_src=01:00:00:00:00:00/01:00:00:00:00:00, actions=drop"
```

- Tiếp đó, switch br0 ở đây cũng không chuyển tiếp gói tin STP chuẩn IEEE 802.1D hoặc địa chỉ MAC đích là địa chỉ reversed multicast.

```sh
ovs-ofctl add-flow br0 \
    "table=0, dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0, actions=drop"
```

- Với các gói tin khác ta coi là hợp lệ thì chuyển gói tin sang bước tiếp theo trên Open Flow table 1:

```sh
ovs-ofctl add-flow br0 "table=0, priority=0, actions=resubmit(,1)"
```

### Kiểm thử Table 0
- Thử với command: `ovs-appctl ofproto/trace br0 in_port=1,dl_dst=01:80:c2:00:00:05`, đầu ra sẽ tương tự như sau:

```sh
Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=01:80:c2:00:00:05,dl_type=0x0000

bridge("br0")
-------------
 0. dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0, priority 32768
    drop

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=01:80:c2:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
Datapath actions: drop
```

Dòng đầu tiên cho biết flow đang duyệt. Nhóm các dòng tiếp theo cho biết hành trình của gói tin qua bridge br0. Open Flow flow table 0 thấy rằng gói tin có địa chỉ đích là địa chỉ reversed multicast và khớp với flow đã thiết lập nên hủy bỏ gói tin.
	
- Thử với một command khác: `ovs-appctl ofproto/trace br0 in_port=1,dl_dst=01:80:c2:00:00:10`, đầu ra sẽ tương tự như sau:

```sh
Flow: in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=01:80:c2:00:00:10,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. No match.
    drop

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=01:80:c2:00:00:10/ff:ff:ff:ff:ff:f0,dl_type=0x0000
Datapath actions: drop
```	

Lúc này, flow xử lý bởi `ofproto/trace` không khớp với bất kì "drop" flow nào trong Table 0 và nó chuyển qua flow có độ ưu tiên thấp hơn là "resubmit" để đưa gói tin sang table 1 xử lý ở chặng tiếp theo.
	
## <a name="tbl1"></a> 5. Triển khai Table 1: VLAN input processing
- Gói tin sau khi đã vượt qua bước xác thực cơ bản ở table 0 sẽ đi vào table 1 để chứng thực VLAN của gói tin dựa trên cấu hình VLAN của port mà gói tin đi qua. Nếu gói tin đi vào access port mà chưa có VLAN header chỉ định thuộc VLAN nào thì nó sẽ được chèn thêm VLAN header để xử lý tiếp.
- Đầu tiên, thực hiện thêm flow mặc định với mức ưu tiếp thấp để hủy bỏ mọi gói tin không khớp flow nào khác:

```sh
ovs-ofctl add-flow br0 "table=1, priority=0, actions=drop"
```
- Tiếp đó, gửi mọi gói tin đi vào port 1 sang table tiếp theo:

```sh
 ovs-ofctl add-flow br0 \
    "table=1, priority=99, in_port=1, actions=resubmit(,2)"
```
- Trên các access port khác, gói tin đi tới mà không có VLAN header sẽ được gắn VLAN number tương ứng với access port, sau đó được chuyển tới bảng tiếp theo:

```sh
ovs-ofctl add-flows br0 - <<'EOF'
table=1, priority=99, in_port=2, vlan_tci=0, actions=mod_vlan_vid:20, resubmit(,2)
table=1, priority=99, in_port=3, vlan_tci=0, actions=mod_vlan_vid:30, resubmit(,2)
table=1, priority=99, in_port=4, vlan_tci=0, actions=mod_vlan_vid:30, resubmit(,2)
EOF
```

### Kiểm thử table 1
- Kiểm thử gói tin trên trunk port (p1): `ovs-appctl ofproto/trace br0 in_port=1,vlan_tci=5`. Kết quả đầu ra sẽ tìm kiếm trên table 0, sau đó resubmit sang table 1 rồi resubmit tiếp tới table 2:

```sh
Flow: in_port=1,vlan_tci=0x0005,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. in_port=1, priority 99
    resubmit(,2)
 2. No match.
    drop

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
Datapath actions: drop
```

- Kiểm thử gói tin hợp lệ trên Access Port:`ovs-appctl ofproto/trace br0 in_port=2`. Ở đây gói tin đi vào port 2 mà không có VLAN header nên sẽ được chèn thêm VLAN header tương ứng port 2 với VLAN ID là 20:

```sh
Flow: in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. in_port=2,vlan_tci=0x0000, priority 99
    mod_vlan_vid:20
    resubmit(,2)
 2. No match.
    drop

Final flow: in_port=2,dl_vlan=20,dl_vlan_pcp=0,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000
Megaflow: recirc_id=0,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
Datapath actions: drop
```

- Kiểm thử gói tin không hợp lệ trên Access Port: `ovs-appctl ofproto/trace br0 in_port=2,vlan_tci=5`. Gói tin ở đây với `Tag Control Information` là 5 đi vào port 2 tương ứng VLAN 20 sẽ bị hủy:

```sh
Flow: in_port=2,vlan_tci=0x0005,dl_src=00:00:00:00:00:00,dl_dst=00:00:00:00:00:00,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. priority 0
    drop

Final flow: unchanged
Megaflow: recirc_id=0,in_port=2,vlan_tci=0x0005,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
Datapath actions: drop
```

## <a name="tbl2"></a> 6. Triển khai Table 2: MAC+VLAN Learning for Ingress Port
- Thêm flow sau:

```sh
ovs-ofctl add-flow br0 \
    "table=2 actions=learn(table=10, NXM_OF_VLAN_TCI[0..11], \
                           NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[], \
                           load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]), \
                     resubmit(,3)"
```

- Ở đây, hành động `learn` chỉnh sửa flow table dựa trên nội dung của flow đang được xử lý. Dưới đây là giải nghĩa các thành phần của hành động `learn`:
	- __table=10__: Chỉnh sửa flow table 10. Đây là MAC learning table.
	- __NXM_OF_VLAN_TCI[0..11]__: đảm bảo flow ta thêm vào flow table 10 sẽ match với cùng VLAN ID của gói tin đang xử lý
	- __NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[]__: đảm bảo flow mà ta vừa thêm vào bảng flow table 10 match địa chỉ Ethernet đích như địa chỉ Ethernet nguồn, điều này chính là việc học địa chỉ MAC của gói tin đi vào từ port nào của bridge, để thực hiện chuyển tiếp một gói tin với đích là MAC vừa học tới port tương ứng với MAC đó.
	- __load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15]__: ghi lại ingress port number vào thanh ghi 0. Thanh ghi 0 sẽ ghi lại địa chỉ port đầu ra mong muốn.

### Kiểm thử table 2:
- Thử command sau:

```sh
ovs-appctl ofproto/trace br0 \
    in_port=1,vlan_tci=20,dl_src=50:00:00:00:00:01 -generate
```	

Kết quả cho biết hành động `learn` được thực thi tại table 2:

```sh
Flow: in_port=1,vlan_tci=0x0014,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. in_port=1, priority 99
    resubmit(,2)
 2. priority 32768
    learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
     -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
    resubmit(,3)
 3. No match.
    drop

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,vlan_tci=0x0014/0x1fff,dl_src=50:00:00:00:00:01,dl_dst=00:00:00:00:00:00/ff:ff:ff:ff:ff:f0,dl_type=0x0000
Datapath actions: drop
```

Xem sự thay đổi trên table 10: `ovs-ofctl dump-flows br0 table=10`:

```sh
NXST_FLOW reply (xid=0x4):
 table=10, vlan_tci=0x0014/0x0fff,dl_dst=50:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
```

Như vậy, gói tin đi tới trên VLAN 20 với MAC nguồn __50:00:00:00:00:01__ sẽ trở thành flow match VLAN 20 với địa chỉ MAC đích là __50:00:00:00:00:01__ trên table 10.


## <a name="tbl3"></a> 7. Triển khai Table 3: Look up destination port
- Table này tìm kiếm xem port nào để gửi gói tin tới dựa trên địa chỉ MAC đích và VLAN.
- Thêm flow để thực hiện tìm kiếm:

```sh
ovs-ofctl add-flow br0 \
    "table=3 priority=50 actions=resubmit(,10), resubmit(,4)"
```

Hành động đầu tiên của flow là resubmit sang table 10. Flows đã học được trong bảng này có ghi port vào thanh ghi 0. Nếu đích của packet chưa được học thì flow matching thất bại. Điều đó cũng có nghĩa là thanh ghi 0 giờ đây vẫn có giá trị là 0 và sẽ là điều kiện để đưa ra tín hiệu flood gói tin ở bước tiếp theo trên table 4.

Hành động thứ hai là resubmit tới table 4 và tiếp tục bước tiếp theo của pipeline.

- Thêm flow khác để bỏ qua tìm kiếm cho gói tin multicast và broadcast:

```sh
ovs-ofctl add-flow br0 \
    "table=3 priority=99 dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 \
      actions=resubmit(,4)"
```

### Kiểm thử table 3
- Đầu tiên, học __f0:00:00:00:00:01__ trên p1 thuộc VLAN 20:

```sh
ovs-appctl ofproto/trace br0 \
    in_port=1,dl_vlan=20,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01 \
    -generate
```

Lúc này địa chỉ MAC đích chưa được học, output sẽ tương tự như sau:

```sh
Flow: in_port=1,dl_vlan=20,dl_vlan_pcp=0,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. in_port=1, priority 99
    resubmit(,2)
 2. priority 32768
    learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
     -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
    resubmit(,3)
 3. priority 50
    resubmit(,10)
    10. No match.
            drop
    resubmit(,4)
 4. No match.
    drop

Final flow: unchanged
Megaflow: recirc_id=0,in_port=1,dl_vlan=20,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01,dl_type=0x0000
Datapath actions: drop
```

Tuy nhiên lúc này source MAC của gói tin đã được học và ghi vào table 10. Thử dump-flows của br0 trên table 10: `ovs-ofctl dump-flows br0 table=10`, kết quả tương tự như sau:

```sh
table=10, vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 actions=load:0x1->NXM_NX_REG0[0..15]
```

- Thực hiện test với gói tin có địa chỉ MAC nguồn và đích đảo ngược lại so với thao tác trên:

```sh
ovs-appctl ofproto/trace br0
in_port=2,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01 -generate
```

Output của lệnh trên sẽ tương tự như dưới đây, với thao tác resubmit(,10) cho thấy rằng gói tin đã match flow đối với địa chỉ MAC đầu tiên đã học được, đồng thời load chỉ số port 0x1 tương ứng với port p1 vào thanh ghi 0 và resubmit sang Table 4 là bước xử lý gói tin sau cùng.

```sh
Flow: in_port=2,vlan_tci=0x0000,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. in_port=2,vlan_tci=0x0000, priority 99
    mod_vlan_vid:20
    resubmit(,2)
 2. priority 32768
    learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
     -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=90:00:00:00:00:01 priority=32768 actions=load:0x2->NXM_NX_REG0[0..15]
    resubmit(,3)
 3. priority 50
    resubmit(,10)
    10. vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01, priority 32768
            load:0x1->NXM_NX_REG0[0..15]
    resubmit(,4)
 4. No match.
    drop

Final flow: reg0=0x1,in_port=2,dl_vlan=20,dl_vlan_pcp=0,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01,dl_type=0x0000
Megaflow: recirc_id=0,in_port=2,vlan_tci=0x0000,dl_src=90:00:00:00:00:01,dl_dst=f0:00:00:00:00:01,dl_type=0x0000
Datapath actions: drop
```

Cũng có thể thấy ở trên kết quả trên rằng ở bước thứ ba, Table 10 đã học được địa chỉ MAC nguồn của gói tin trên port 2 và load chỉ số port tương ứng là 0x2 vào thanh ghi 0. Như vậy, Table 10 đã học được cả hai địa chỉ MAC nguồn và đích của gói tin ở bước kiểm thử đầu tiên.

- Tiếp theo, kiểm tra lại gói tin như bước kiểm thử đầu:

```sh
ovs-appctl ofproto/trace br0 \
in_port=1,dl_vlan=20,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01 -generate
```

Kết quả đầu ra cho thấy ở bước thứ 4 này, bridge đã tìm thấy được MAC đích của gói tin bằng việc tìm kiếm trên table 10 và resubmit sang Table 4 để xử lý tiếp:

```sh
Flow: in_port=1,dl_vlan=20,dl_vlan_pcp=0,dl_src=f0:00:00:00:00:01,dl_dst=90:00:00:00:00:01,dl_type=0x0000

bridge("br0")
-------------
 0. priority 0
    resubmit(,1)
 1. in_port=1, priority 99
    resubmit(,2)
 2. priority 32768
    learn(table=10,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:NXM_OF_IN_PORT[]->NXM_NX_REG0[0..15])
     -> table=10 vlan_tci=0x0014/0x0fff,dl_dst=f0:00:00:00:00:01 priority=32768 actions=load:0x1->NXM_NX_REG0[0..15]
    resubmit(,3)
 3. priority 50
    resubmit(,10)
    10. vlan_tci=0x0014/0x0fff,dl_dst=90:00:00:00:00:01, priority 32768
            load:0x2->NXM_NX_REG0[0..15]
    resubmit(,4)
 4. No match.
    drop
```

## <a name="tbl4"></a> 8. Triển khai Table 4: Output Processing
- Tại entry của bước 4, ta biết rằng thanh ghi 0 sẽ chứa port mà gói tin sẽ được chuyển tiếp tới hoặc bằng 0 để thực hiện flood. 
- Ở bước cuối cùng của pipeline, gói tin sẽ thực sự được xử lý tới đầu ra phù hợp. Đầu tiên là đẩy gói tin ra khỏi trunk port p1:

```sh
ovs-ofctl add-flow br0 "table=4 reg0=1 actions=1"
```

- Đối với output ra các access port, ta thực hiện loại bỏ VLAN header trước đi đẩy gói tin ra:

```sh
ovs-ofctl add-flows br0 - <<'EOF'
table=4 reg0=2 actions=strip_vlan,2
table=4 reg0=3 actions=strip_vlan,3
table=4 reg0=4 actions=strip_vlan,4
EOF
```

- Đối với các gói tin broadcast và multicast với MAC đích không cụ thể thì thực hiện flood gói tin ra trunk port (với cả 802.1Q header) và gỡ bỏ VLAN header trước khi gửi ra các access port:

```sh
ovs-ofctl add-flows br0 - <<'EOF'
table=4 reg0=0 priority=99 dl_vlan=20 actions=1,strip_vlan,2
table=4 reg0=0 priority=99 dl_vlan=30 actions=1,strip_vlan,3,4
table=4 reg0=0 priority=50            actions=1
EOF
```

### Kiểm thử Table 4
- Bài test 1: Broadcast, Multicast, and Unknown Destination
	- Duyệt gói broadcast với VLAN ID 30 trên port p1:

    ```sh
    ovs-appctl ofproto/trace br0 \
    	in_port=1,dl_dst=ff:ff:ff:ff:ff:ff,dl_vlan=30
    ```

	- Kết quả đầu ra sẽ là flood gói tin ra hai cổng p3 và p4 với VLAN header bị loại bỏ:

    ```sh
    Datapath actions: pop_vlan,3,4
    ```

	- Tương tự với gói tin broadcast tới p3, gói tin đi tới chưa có VLAN header sẽ được đóng VLAN header với ID 30 rồi flood qua trunk port p1 và access port p4:

    ```sh
    ovs-appctl ofproto/trace br0 in_port=3,dl_dst=ff:ff:ff:ff:ff:ff
    ```
	
	Với output:

    ```sh
    Datapath actions: push_vlan(vid=30,pcp=0),1,pop_vlan,4
    ```
	
	- Các gói tin broadcast bị loại bỏ do nó chỉ thuộc về input port mà thôi:

    ```sh
    ovs-appctl ofproto/trace br0 \
    	in_port=1,dl_dst=ff:ff:ff:ff:ff:ff  

    ovs-appctl ofproto/trace br0 \
    	in_port=1,dl_dst=ff:ff:ff:ff:ff:ff,dl_vlan=55
    ```
	
- Bài test 2: MAC learning
	- Đầu tiên, học MAC của gói tin thuộc VLAN 30 trên port p1:

    ```sh
    ovs-appctl ofproto/trace br0 \
    	in_port=1,dl_vlan=30,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01 \
    	-generate
    ```
	
	- Có thể thấy rằng gói tin này sẽ được flood trên hai cổng p3 và p4 thuộc VLAN 30, đồng thời cũng học được địa chỉ MAC nguồn của gói tin tới trên port 1:

    ```sh
    Datapath actions: pop_vlan,3,4
    ```
	
	- Tận dụng địa chỉ MAC đã học được đó, ta gửi một gói tin qua port 4 với các địa chỉ MAC đảo ngược với gói trên 

    ```sh
    ovs-appctl ofproto/trace br0 \
    	in_port=4,dl_src=20:00:00:00:00:01,dl_dst=10:00:00:00:00:01 -generate
    ```
	
	- Lúc này bridge tìm kiếm trên table 10 thấy được MAC đích đã học được từ MAC nguồn của gói tin trước nên chuyển tiếp qua cổng 1 mà không flood thêm qua cổng 3 nữa:

    ```sh
    Datapath actions: push_vlan(vid=30,pcp=0),1
    ```
	
	- Tương tự như vậy, kiểm tra lại gói ban đầu, ta thấy rằng nó cũng không còn flood qua cổng 3 nữa mà được loại bỏ VLAN header và gửi qua cổng p4:

    ```sh
    ovs-appctl ofproto/trace br0 \
    	in_port=1,dl_vlan=30,dl_src=10:00:00:00:00:01,dl_dst=20:00:00:00:00:01 \
    	-generate
    ...
    Datapath actions: pop_vlan,4		
    ```	