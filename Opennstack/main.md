

## 2. Kiến trúc node trong OpenStack
OpenStack chia các chức năng ra nhiều node để tách biệt tải và trách nhiệm:

### Controller Node
- Điều khiển toàn bộ cluster, chạy tất cả các API service và management:
  - Keystone — xác thực, token
  - Glance — quản lý image
  - Nova API — nhận request tạo VM
  - Neutron Server — quản lý network logic
  - Horizon — dashboard UI
  - Database (MariaDB) + Message Queue (RabbitMQ)
- Các process chạy trên controller node
  - keystone          → Auth & token
  - glance-api        → Image management
  - nova-api          → VM lifecycle API
  - nova-scheduler    → Quyết định VM chạy ở Compute nào
  - nova-conductor    → Trung gian Nova API ↔ DB
  - neutron-server    → Network API
  - cinder-api        → Block storage API
  - cinder-scheduler  → Quyết định volume tạo ở Storage nào
  - horizon           → Dashboard UI
  - mariadb / galera  → Database (clustered)
  - rabbitmq          → Message queue
  - memcached         → Token cache
  - haproxy           → Load balancer cho các API
  - keepalived        → VIP failover (HA)


### Compute Node
- Là nơi VM thật chạy trực tiếp:
- Nova Compute nhận lệnh từ Controller, spawn VM via libvirt/KVM
- OVS/OVN agent — kết nối VM vào network
- Càng nhiều Compute node → càng chạy được nhiều VM.
- Các process chạy trên compute note
  - nova-compute      → Spawn/stop VM (libvirt/KVM)
  - neutron-agent     → Kết nối VM vào virtual network
  - ovn-controller    → OVN local agent
  - openvswitch       → Virtual switch cho VM traffic
  - libvirtd          → Hypervisor daemon

### Network Node
- Xử lý traffic ra vào (North-South) và giữa các VM:
  - Neutron agents — L3 router, DHCP, metadata
  - OVN controller
  - Floating IP, NAT
- Trong setup nhỏ thường gộp vào Controller. Tách riêng khi traffic lớn.
- Cacs process chạy trên network node
  - neutron-l3-agent          → Virtual router, NAT
  - neutron-dhcp-agent        → Cấp IP cho VM
  - neutron-metadata-agent    → Cloud-init metadata cho VM
  - ovn-controller            → Distributed virtual networking
  - openvswitch               → Virtual switch

### Storage Node (tuỳ chọn)
- Cung cấp block storage và object storage:
  - Cinder — block storage (giống EBS của AWS)
  - Swift / Ceph — object storage (giống S3)
- Các process chạy trên storage node
  - cinder-volume     → Tạo/attach block volume
  - ceph-osd          → Object storage daemon (nếu dùng Ceph)
  - ceph-mon          → Ceph monitor

## 3. Networking trong Openstack
### 3.1 Network
- Là một đối tượng trừu tượng đại diện cho một broadcast domain Layer 2 — tương đương với một VLAN hoặc một switch ảo.
- Khi tạo một network, Neutron thực chất tạo ra một phân đoạn mạng riêng biệt mà các máy ảo (VM) có thể kết nối vào. Mỗi network được gán một segmentation ID (tunnel key) để cô lập traffic giữa các network với nhau trên cùng hạ tầng vật lý
- Một network có thể thuộc các loại khác nhau tuỳ thuộc vào cách triển khai phía dưới: VXLAN, VLAN, GRE, hay flat.
- Network tự nó chưa có thông tin IP — nó chỉ là "dây dẫn". Để có địa chỉ IP, cần tạo subnet gắn vào network đó (ví dụ 10.0.0.0/24). Một network có thể có nhiều subnet với dải IP khác nhau vẫn hợp lệ
- Tất cả các VM, router, agent thuộc cùng network có thể giao tiếp Layer 2 với nhau
- Các loại Network:
  - Provider Network: Được tạo bởi admin, ánh xạ trực tiếp đến một hạ tầng mạng vật lý có sẵn. VM gắn vào provider network có thể dùng floating IP hoặc fixed IP trực tiếp
  - Project/Tenant Network: Được tạo bởi user thường, hoàn toàn ảo, isolated giữa các project.
  - External Network: Một loại network đặc biệt, đại diện cho mạng bên ngoài (internet hoặc corporate network). Router ảo dùng network này làm gateway (SNAT ra ngoài) Floating IP được cấp phát từ pool của external network

**Luồng traffic điển hình**

<img width="535" height="418" alt="image" src="https://github.com/user-attachments/assets/3627254b-f863-424e-af29-933a24d520b2" />

### 3.2. Port
- Là một điểm kết nối logic vào network — tương đương với một cổng trên switch ảo. Mỗi port có:
  - Một MAC address duy nhất
  - Một hoặc nhiều fixed IP (lấy từ subnet của network mà port thuộc về)
  - Các security group rules áp dụng cho traffic đi qua port
  - Một binding profile cho biết port đang được gắn vào đâu (compute host nào, dùng OVN/OVS nào)
- Một network có thể chứa nhiều port, mỗi port chỉ thuộc về đúng một network.
- Hình dung đơn giản: network là switch ảo, còn port là cổng trên switch đó. Khi VM, router, DHCP agent, hay load balancer cần kết nối vào một mạng, chúng đều làm điều đó thông qua một port. Cụ thể:
  - VM gắn vào network → Neutron tạo port loại compute:nova
  - Router kết nối vào network → Neutron tạo port loại network:router_interface (hoặc network:router_gateway cho external network)
  - DHCP agent phục vụ subnet → Neutron tạo port loại network:dhcp
- Port cũng có thể được tạo thủ công bằng Neutron API
- Flow hoạt động khi boot một VM:
  - Neutron tạo một port trên network VM thuộc về, cấp MAC + IP,. Lúc này Neutron port chỉ là bản ghi logical trong database, chưa có gì trên host
  - OVN controller trên compute node watch thông tin từ OVN Northbound DB, tạo ra OVS port thực sự trên br-int của host. Virtual NIC của VM (tap interface) được map vào OVS port

### 3.3 Bridge
- Bridge là switch ảo nối nhiều network interface lại với nhau thành một L2 domain duy nhất. Nó nhận các interface (vật lý hoặc ảo) làm "port" và chuyển tiếp frame Ethernet giữa chúng dựa trên MAC address.
- Nhiệm vụ của bridge:
  - Switching — VM1 gửi frame đến VM2, bridge tra MAC table, forward thẳng sang tap-vm2 mà không broadcast ra các port khác.
  - Điểm kết nối tunnel — interface VXLAN/GRE cũng cắm vào bridge, nên frame từ VM1 trên node này có thể đi qua tunnel sang bridge cùng tên trên node khác, rồi đến VM4.
  - Điểm áp dụng security group — với Linux Bridge agent, iptables rules được đặt trên tap interface trước khi frame vào bridge, để filter traffic theo security group của từng VM.
- Lưu ý khi VM muốn giao tiếp với VM ở subnet khác, packet phải rời bridge, đi lên router (qrouter namespace, OVN logical router), rồi mới được forward sang subnet đích.
- Trong Openstack có 3 Mechanism Drivers sử dụng bridge:
  - linuxbridge driver: sử dụng Linux Bridge của kernal. Mỗi Neutron network tạo ra một bridge riêng ( có dạng `brq-<net-id>`), dùng iptables/ebtables cho security group. Phù hợp môi trường nhỏ, dễ debug bằng brctl/ip link.
  - openvswitch driver: dùng Open vSwitch với OpenFlow rules. Một br-int duy nhất cho tất cả VM, br-tun xử lý tunnel riêng, br-ex kết nối ra ngoài. Mạnh hơn nhưng phức tạp hơn để debug.
  - ovn driver: vẫn sử dụng OVS bridge bên dưới, nhưng bỏ br-tun đi. OVN Controller tự lập trình Geneve tunnel thẳng vào br-int. Logic L2/L3 được định nghĩa tập trung trong Northbound DB rồi compile xuống, routing phân tán (distributed) mặc định.
  - Còn 2 driver macvtap và sriovnicswitch không sử dụng bridge



#### 3.3.1. Linuxbridge driver
  <img width="670" height="324" alt="image" src="https://github.com/user-attachments/assets/3ced1bba-1f2d-4b26-86b5-ce39c7f0ee0d" />
  
- Với linuxbridge driver, trên mỗi compute node sẽ có 1 Linux Bridge agent. Khi tạo các VM nằm trên nhiều compute node nhưng cùng thuộc network A thì Linux Bridge agent trên các node có VMs sẽ tự động tạo bridge trên compute node bằng lệnh `ip link add brq-xxx type bridge`. Các VM cùng thuộc một network trên cùng một node sẽ cắm vào chung một bridge đấy. Và khi khởi tạo VM, kernel tạo một tap interface và cắm vào bridge. Từ góc nhìn của VM, nó đang cắm vào một switch — không biết và không cần biết bên dưới là Linux bridge hay OVS.
- eth0 và tap interface là 2 đầu của 1 đường ống, trong VM chỉ thấy eth0 (hoặc ens3, ens4... tùy distro), còn tap interface là đầu nối phía host, chỉ nhìn thấy trên host vật lý
- Minh họa: cả hai node đều có brq-net-A riêng. VM1, VM2 cắm vào bridge trên chính node của chúng. Hai bridge được nối với nhau qua VXLAN tunnel — nhờ đó VM1 và VM3 tuy ở hai máy vật lý khác nhau nhưng vẫn thấy nhau như cùng một mạng L2.
```
Compute node 1                    Compute node 2
┌─────────────────┐               ┌─────────────────┐
│  brq-net-A      │               │  brq-net-A      │
│  ┌───┐  ┌───┐   │               │  ┌───┐          │
│  │VM1│  │VM2│   │               │  │VM3│          │
│  └───┘  └───┘   │               │  └───┘          │
│  vxlan──────────┼───────────────┼──vxlan          │
└─────────────────┘               └─────────────────┘
```

#### 3.3.2. openvswitch driver

<img width="672" height="565" alt="image" src="https://github.com/user-attachments/assets/a131a69d-c2e8-4815-a813-9ca08a70976c" />

- openvswitch driver sử dụng 4 loại OVS bridge:
  - br-int (Integration Bridge) làm bridge trung tâm, mọi VM đều gắn vào đây thông qua tap interface. Traffic được phân tách bằng internal VLAN tag. Lưu ý OVS không dùng VLAN tag của tenant network trực tiếp — nó tự tạo ra internal VLAN (local VLAN) riêng cho mỗi host. Ví dụ: network "tenantA" có thể là VLAN 5 ở host 1 nhưng VLAN 9 ở host 2. OVS flows trong br-int xử lý việc map giữa tenant network ID ↔ local VLAN
  - br-tun (Tunnel Bridge) khi cần thiết lập dùng overlay network (VXLAN, GRE). Mỗi compute node có một br-tun, chúng kết nối peer-to-peer hoặc qua multicast. br-tun sẽ chuyển đổi local VLAN ↔ VNI (VXLAN Network Identifier). Khi packet rời host, br-tun strip local VLAN và đóng gói VXLAN với VNI tương ứng. Khi packet đến, nó decap VXLAN và gán lại local VLAN phù hợp với host đó. Hai bridge nói chuyện với nhau qua patch port — một loại virtual port nội bộ của OVS, không phải physical interface.
  - br-ex (External Bridge) để ra mạng vật lý bên ngoài. Router ảo (qrouter-* namespace) có một chân (qr-xxx) cắm vào br-int và một chân (qg-xxx) cắm vào br-ex. Khi VM có floating IP ra internet, packet đi qua qrouter để DNAT/SNAT rồi ra br-ex.
  - OVS br-provider (optional) được sử dụng khi có provider network (flat/VLAN trực tiếp từ vật lý). Bypass tunnel, traffic đi thẳng ra physical switch với VLAN tag thật

- Nhược điểm openvswitch so với ovn là mỗi Neutron agent phải tự viết OpenFlow rules trên từng host — không có cơ chế tập trung, dễ mất đồng bộ khi scale lớn.
- Luồng traffic:
  - giữa 2 VM cùng host, cùng network
  <img width="656" height="62" alt="image" src="https://github.com/user-attachments/assets/cb0ed29f-7b7a-42a7-a914-2610e8ccf050" />
  
  - giữa 2 VM khác host, cùng network
  <img width="634" height="135" alt="image" src="https://github.com/user-attachments/assets/578ba4fd-b907-4bb1-afe5-a372cf1fa38a" />
  
  - Từ VM đi ra internet (floating IP)
  <img width="684" height="62" alt="image" src="https://github.com/user-attachments/assets/26401a7e-80b5-4243-9b96-c0f6a857907f" />


Note: br-int/br-tun/br-ex/br-provider đều là OVS switch, chỉ khác nhau ở loại port được gắn vào và OpenFlow rules được lập trình:. Có thể kiểm tra trực tiếp trên host bằng lệnh `ovs-vsctl show` -> Output:
  ```
  Bridge br-int
      fail_mode: secure
      Port br-int
          Interface br-int
              type: internal
      Port tap_aaa
          Interface tap_aaa
      Port patch-tun
          Interface patch-tun
              type: patch
              options: {peer=patch-int}
  
  Bridge br-tun
      Port patch-int
          Interface patch-int
              type: patch
              options: {peer=patch-tun}
      Port vxlan-c0a80102
          Interface vxlan-c0a80102
              type: vxlan
              options: {remote_ip="192.168.1.2"}
  
  Bridge br-ex
      Port br-ex
          Interface br-ex
              type: internal
      Port eth1
          Interface eth1
      Port qg-xxxx
          Interface qg-xxxx
  ```

#### 3.3.3. ovn driver
- ovn driver vẫn dùng OVS bridge, nhưng đơn giản hơn so với openvswitch driver vì đã loại bỏ br-tun
VM
 └─ tap_xxx
      └─ br-int          ← Integration bridge (OVS)
            └─ geneve0   ← Tunnel port do OVN Controller tạo tự động

br-ex                    ← External bridge (OVS) — kết nối ra ngoài
Không còn br-tun nữa — đây là điểm khác biệt lớn nhất.

Tại sao OVN bỏ được br-tun?
Với OVS legacy, br-tun tồn tại vì:

Neutron agent cần một bridge riêng để xử lý tunnel (VXLAN/GRE)
Logic chuyển đổi local VLAN ↔ VNI được viết bằng OpenFlow trên br-tun

Với OVN:

OVN Controller (ovn-controller) tự lập trình Geneve tunnel trực tiếp vào br-int
Không cần bridge trung gian
Logic L2/L3 được định nghĩa ở Logical Switch / Logical Router trong OVN Northbound DB, rồi compile xuống OpenFlow


Kiến trúc đầy đủ
┌─────────────────────────────────────────┐
│  OVN Northbound DB                      │
│  (Logical Switch, Logical Router, ACL)  │
└──────────────┬──────────────────────────┘
               │ ovn-northd compile
┌──────────────▼──────────────────────────┐
│  OVN Southbound DB                      │
│  (Logical Flow, Binding, MAC binding)   │
└──────────────┬──────────────────────────┘
               │ ovn-controller đọc & lập trình
┌──────────────▼──────────────────────────┐
│  OVS (trên mỗi compute node)            │
│                                         │
│  br-int                                 │
│   ├─ tap_vm1, tap_vm2, ...              │
│   └─ geneve port (tunnel tới node khác) │
│                                         │
│  br-ex                                  │
│   ├─ patch port tới br-int              │
│   └─ physical NIC (eth1)               │
└─────────────────────────────────────────┘

So sánh nhanh số lượng bridge
DriverBridges cần cóLinux Bridgebrq-<id> × N network + vxlan interfaceOVS Legacybr-int + br-tun + br-exOVNbr-int + br-ex

br-ex trong OVN có gì đặc biệt?
OVN dùng khái niệm "localnet port" — một logical port đặc biệt nối Logical Switch với physical network thông qua br-ex.
OVN Logical Switch
 └─ localnet port "physnet1"
      └─ map tới br-ex (qua ovn-bridge-mappings)
            └─ eth1 (physical NIC)
Cấu hình trên mỗi node:
bashovs-vsctl set open . \
  external-ids:ovn-bridge-mappings="physnet1:br-ex"

Tóm lại: OVN vẫn là OVS bridge, chỉ là ít bridge hơn và logic được quản lý tập trung thay vì mỗi agent tự viết flows. 

==========================

<img width="801" height="485" alt="image" src="https://github.com/user-attachments/assets/7fb7196d-4f15-4d01-b385-13de01f15b1f" />

#### Điểm khác biệt giữa L3 Neutron và OVN backend network
##### L3 Neutron



##### OVN
- ovn-controller tạo br-int ngay khi service khởi động, không cần đợi VM nào. Toàn bộ vòng đời của VM chỉ liên quan đến việc thêm/xóa port trên br-int và cập nhật OVS flow rules — không bao giờ tạo thêm bridge mới.
br-ex là ngoại lệ — không do agent tạo động, mà được tạo sẵn lúc cài đặt OpenStack (Kolla, manual...) và gán physical interface vào đó một lần duy nhất. Đó là lý do bạn thấy enp1s0 nằm trong br-ex khi chạy ovs-vsctl show.
- Khi gõ lệnh `ovs-vsctl show` trên compute node sẽ chỉ thấy 1 br-int và mọi VM tap interface đều kết nối vào đó. OVS gán tunnel key (còn gọi là VNI — Virtual Network Identifier) cho các packets đến từ các network khác nhau (VD Net-A gán tunnel key 100, Net-B gán tunnel key 200). OVS flow tables trong kernel có rule để xử lý dựa trên tunnel key
## 4. Neutron

<img width="701" height="590" alt="image" src="https://github.com/user-attachments/assets/b7160aff-001d-4047-9e1e-20ff47ce3d0b" />

Giải thích các khái niệm
- ML2 Plugin (Modular Layer 2): là core plugin duy nhất của Neutron hiện tại, thay thế các monolithic plugin cũ. "Modular" vì nó cho phép kết hợp linh hoạt các driver. ML2 gồm 3 loại driver hoạt động độc lập với nhau:
  - Type Drivers: định nghĩa loại network vật lý được dùng để vận chuyển traffic. flat là mạng không có VLAN tag, vlan dùng 802.1Q, vxlan/geneve/gre là các overlay tunnel protocol. Setup của bạn với OVN thường dùng geneve.
  - Mechanism Drivers — định nghĩa cách implement network đó trên thực tế (ai lập trình switch ảo, ai xử lý routing). Đây chính là thứ người ta hay gọi không chính thức là "backend" hoặc "ML2 driver". Ba tên gọi này đều chỉ cùng một thứ. Mechanism Driver được cấu hình ở dòng `mechanism_drivers =` trong ml2_conf.ini.
  - Extension Drivers — các tính năng tùy chọn gắn thêm vào port/network như QoS, port security, DNS. Không liên quan đến data plane.

### 4.1. Mechanism Drivers (backend/ML2 driver)
- Chức năng: Biến "khai báo mạng" thành "mạng thật sự hoạt động". Khi chạy các lệnh như `openstack network create my-net`, `openstack router create my-router`, `openstack server create --network my-net my-vm`, Neutron chỉ lưu thông tin vào database nhưng chưa thực sự cấu hình ở tầng hệ điều hành và phần cứng. Backend networking là thứ biến các bản ghi trong DB đó thành hạ tầng mạng thật sự.
- Cụ thể công việc của backend networking khi bạn tạo một VM và gán vào network:
  - Trên compute node: tạo virtual port, kết nối VM vào virtual switch (OVS bridge hoặc Linux bridge), áp dụng security group rules (iptables hoặc OVS flows) để filter traffic của VM đó.
  - Trên network node: tạo router ảo (qrouter namespace với L3 Agent, hoặc logical router trong OVN), cấu hình routing giữa các subnet, setup NAT cho floating IP, chạy DHCP server để VM lấy được IP.
  - Giữa các node: tạo tunnel (Geneve/VXLAN) để traffic của VM trên compute01 có thể đi sang VM trên compute02, dù hai VM đang ở hai máy vật lý khác nhau.
- Có các type phổ biến là OVN, openvswitch (mặc định của Neutron) và SR-IOV
- Hầu hết các installer hiện đại (Kolla-AnsibleOVN, OpenStack-AnsibleOVN, Canonical MicroStack,...) đều mặc định dùng OVN vì đó là hướng phát triển chính của OpenStack từ khoảng bản Yoga (2022) trở đi

#### 4.1.1. Neutron truyền thống (L3 Agent)
<img width="788" height="530" alt="image" src="https://github.com/user-attachments/assets/e6fe716b-3d02-4c76-93ad-f9778025bedb" />

- Neutron truyền thống dùng các agent chạy ở userspace trên Network node. Khi bạn tạo 1 virtual router, L3 agent sẽ tạo ra 1 Linux network namespace (qrouter-xxx) với iptables rules và ip route riêng. 10 router = 10 namespace = 10 iptables chain. Traffic đi qua userspace nên chậm hơn và tốn CPU.
- Mỗi virtual router tạo ra một qrouter-* namespace độc lập trên Network node, mỗi network tạo ra một qdhcp-* namespace. 100 router = 100 namespace.
- Routing xử lý ở userspace (iptables), tốn tài nguyên và khó scale.

#### 4.1.2. OVN (Open Virtual Network)

<img width="800" height="523" alt="image" src="https://github.com/user-attachments/assets/f8c0ca68-9c2a-4e8c-83e1-223b84f509b4" />

- ovn-northd trên Controller đọc logical config từ NB DB (Northbound Database), dịch nó thành OVS flow rules, rồi ghi vào SB DB (Southbound Database).
- Sau đó ovn-controller trên mỗi node (network01, compute01) đọc SB DB và lập trình trực tiếp vào OVS kernel flow tables.
- Không có namespace, không có iptables — routing xảy ra trong kernel thông qua OVS, nhanh hơn nhiều.

#### 4.1.3. SR-IOV
- Dùng khi cần performance cực cao (telco, HPC). VM được cấp virtual function (VF) của NIC vật lý, bypass hoàn toàn OVS/kernel networking.
- Latency rất thấp nhưng mất tính flexibility (không dùng được floating IP, security group giới hạn).


## 5. Các lệnh làm việc với Openstack

### 4.1 Compute (Nova)
- Liệt kê instances
  - `openstack server list --all-projects`
  - `openstack server list --project <project_id>`
- Tạo instance
  - `openstack server create --flavor m1.small --image ubuntu-22.04 --network <network-name> --key-name my-key --security-group default my-instance`
- Thao tác với instance
  - `openstack server show <server-id>`
  - `openstack server start/stop/reboot <server-id>`
  - `openstack server delete <server-id>`
  - `openstack server suspend/resume <server-id>`
  - `openstack server pause/unpause <server-id>`
- Console / log
  - `openstack console url show <server-id> --os-project-name <tên-project>`
  - `openstack console log show <server-id>`
- Migration
  - `openstack server migrate <server-id>`
  - `openstack server migrate --live <host> <server-id>`
- Resize
  - `openstack server resize --flavor m1.large <server-id>`
  - `openstack server resize confirm <server-id>`
- Liệt kê hypervisors
  - `openstack hypervisor list`
  - `openstack hypervisor show <hypervisor>`
  - `openstack hypervisor stats show`

### 4.2. Network (Neutron)
- Liệt kê và tạo network
  - `openstack network list`
  - `openstack network create <name> [--external] [--provider-network-type flat]`
  - `openstack network delete <name>`
- Liệt kê và tạo subnet
  - `openstack subnet list`
  - `openstack subnet create --network <network> --subnet-range 192.168.1.0/24 --gateway 192.168.1.1 --dns-nameserver 8.8.8.8 my-subnet`
- Liệt kê và tạo Routers
  - `openstack router list`
  - `openstack router create my-router`
  - `openstack router set --external-gateway <ext-network> my-router`
  - `openstack router add subnet my-router my-subnet`
  - `openstack router remove subnet my-router my-subnet`
- Floating IPs
  - `openstack floating ip list`
  - `openstack floating ip create <ext-network>`
  - `openstack floating ip set --port <port-id> <floating-ip>`
  - `openstack server add floating ip <server-id> <floating-ip>`
  - `openstack server remove floating ip <server-id> <floating-ip>`
- Liệt kê và tạo Security Groups
  - `openstack security group list`
  - `openstack security group create my-sg`
  - `openstack security group rule create --protocol tcp --dst-port 22 --remote-ip 0.0.0.0/0 my-sg`
  - `openstack security group rule list my-sg`
- Liệt kê và tạo Ports
  - `openstack port list --server <server-id>`
  - `openstack port show <port-id>`

### 4.3. Storage (Cinder & Swift)
- Liệt kê và tạo Volumes (Cinder)
  - `openstack volume list`
  - `openstack volume create --size 20 --type <type> my-volume`
  - `openstack volume show <volume-id>`
  - `openstack volume delete <volume-id>`
- Attach/Detach
  - `openstack server add volume <server-id> <volume-id>`
  - `openstack server remove volume <server-id> <volume-id>`
- Snapshots
  - `openstack volume snapshot create --volume <vol-id> my-snap`
  - `openstack volume snapshot list`
  - `openstack volume snapshot delete <snap-id>`
- Volume types
  - `openstack volume type list`
- Object Storage (Swift)
  - `openstack container list`
  - `openstack container create my-bucket`
  - `openstack object list my-bucket`
  - `openstack object upload my-bucket /path/to/file`
  - `openstack object save my-bucket <object> --file /tmp/out`

### 4.4 Images (Glance)
- Liệt kê images
  - `openstack image list`
  - `openstack image show <image-id>`
- Upload image
  - `openstack image create --file ubuntu-22.04.img --disk-format qcow2 --container-format bare --public ubuntu-22.04`
- Download & delete
  - `openstack image save --file /tmp/out.qcow2 <image-id>`
  - `openstack image delete <image-id>`
- Chia sẻ image giữa projects
  - `openstack image add project <image-id> <project-id>`
  - `openstack image set --shared <image-id>`

### 4.5 Identity (Keystone)
- Projects
  - `openstack project list`
  - `openstack project create <name> --domain Default`
  - `openstack project delete <name>`
- Users
  - `openstack user list`
  - `openstack user create --password <pass> --email x@x.com <name>`
  - `openstack user delete <user>`
- Roles
  - `openstack role list`
  - `openstack role add --user <user> --project <project> member`
  - `openstack role assignment list --project <project>`

### 4.6 Theo dõi & Debug
- Quota
  - `openstack quota show <project>`
  - `openstack quota set --instances 50 --cores 200 --ram 204800 <project>`
- Usage
  - `openstack usage list`
  - `openstack usage show --start 2025-01-01 --end 2025-12-31`
- Services & Agents
  - `openstack service list`
  - `openstack compute service list`
  - `openstack network agent list`
  - `openstack volume service list`
- Kiểm tra logs (trên node)
  - `journalctl -u nova-api -f`
  - `journalctl -u neutron-server -f`
  - `journalctl -u cinder-api -f`
  - `tail -f /var/log/nova/nova-api.log`
  - `tail -f /var/log/neutron/neutron-server.log`
- Endpoints
  - `openstack endpoint list`
  - `openstack endpoint show <endpoint-id>`

### 4.7. Tips hữu ích
- Filter
  - `openstack server list --status ERROR`
  - `openstack server list --host <compute-node>`
  - `openstack volume list --status error`
- Bulk delete instances lỗi
  - `openstack server list --status ERROR -f value -c ID | xargs -I{} openstack server delete {}`


---

=============================================================


Tìm file RC: `find ~/ /etc -name "*rc*" -o -name "*openrc*" 2>/dev/null`

`source /etc/kolla/admin-openrc.sh` -> Lệnh này load các biến môi trường cần thiết để xác thực với OpenStack API vào shell hiện tại. Sau khi chạy lệnh này, bạn có thể dùng được các lệnh OpenStack CLI mà không cần truyền thủ công --os-username, --os-password...:

Ví dụ
```
openstack server list
openstack network list
openstack image list
nova list
neutron net-list
```

File admin-openrc.sh Thường có dạng như sau:
```
bashexport OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=<your_password>
export OS_AUTH_URL=http://<controller_ip>:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

`openstack token issue` -> lấy token. Khi source admin-openrc.sh chỉ load credentials vào biến môi trường, nhưng chưa thực sự gửi request tới Keystone để lấy token. Thực tế khi dùng CLI thông thường sau khi source, bạn không cần openstack token issue cho các lệnh bình thường. Lấy token chi để kiểm tra kết nối

---

Catalog (Service Catalog) trong Openstack là một danh mục các dịch vụ và endpoints của OpenStack, được quản lý bởi Keystone. Khi bạn xác thực thành công, Keystone trả về token kèm theo catalog để client biết gọi API ở đâu. Client gọi thẳng service đó, không qua Keystone nữa

Lệnh `openstack catalog list` trả về output dạng:
```
+-----------+-----------+----------------------------------------------+
| Name      | Type      | Endpoints                                    |
+-----------+-----------+----------------------------------------------+
| keystone  | identity  | RegionOne public: http://10.0.0.1:5000/v3   |
| nova      | compute   | RegionOne public: http://10.0.0.1:8774/v2.1 |
| neutron   | network   | RegionOne public: http://10.0.0.1:9696/v2.0 |
| glance    | image     | RegionOne public: http://10.0.0.1:9292      |
| cinder    | volume    | RegionOne public: http://10.0.0.1:8776/v3   |
+-----------+-----------+----------------------------------------------+
```

Xem endpoint của một service cụ thể `openstack catalog show nova`

Mỗi entry trong catalog gồm 3 thành phần:
```
Service (nova)
    └── Type (compute)
            └── Endpoints
                    ├── public    → URL dùng từ ngoài
                    ├── internal  → URL dùng nội bộ giữa các service
                    └── admin     → URL dùng cho admin operations
```

Catalog giúp các service không cần hardcode URL của nhau. Thay đổi IP/port chỉ cần update catalog trong Keystone, các service tự tìm lại

---

Project trong OpenStack (trước đây gọi là Tenant) là đơn vị tổ chức và phân tách tài nguyên trong OpenStack. Mọi resource (VM, network, volume...) đều thuộc về một project. Project giống như một "không gian làm việc riêng" — các team/khách hàng khác nhau có project riêng, tài nguyên không ảnh hưởng lẫn nhau.

Mối quan hệ Project - User - Role:
- Một user có thể thuộc nhiều project với role khác nhau
- Một project có thể có nhiều user
- Role quyết định user được làm gì trong project đó


Domain là cấp cao hơn, nhóm các project lại

- Liệt kê tất cả project: openstack project list
- Tạo project mới: openstack project create --domain Default --description "Backend Team" team-backend
- Xem chi tiết project: openstack project show team-backend
- Thêm user vào project với role: openstack role add --project team-backend --user alice member
- Xem quota của project: openstack quota show team-backend
- Xóa project: openstack project delete team-backend

---


Kiểm tra VM trên Compute Node của OpenStack
1. Xác định VM đang ở node nào (từ Controller): `openstack server show <vm-name-or-id> | grep -E "host|hypervisor|node"`

Output sẽ cho biết VM đang nằm ở compute node nào:
```
| OS-EXT-SRV-ATTR:host                | compute-node-01          |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute-node-01.local    |
```
2. Xem danh sách tất cả VM và node của chúng
- List tất cả server kèm host `openstack server list --all-projects --long -c Name -c Status -c Host`
- Sau khi biết VM ở node nào, SSH vào compute node đó và kiểm tra bằng virsh (libvirt/KVM) `sudo virsh list --all`. Output sẽ thấy instance tương ứng

---

Neutron sử dụng kiến trúc plugin, chia làm 2 lớp chính: Core Plugin và Service Plugin.

1. Core Plugin (ML2 - Modular Layer 2)
ML2 là core plugin tiêu chuẩn hiện nay, gồm 2 loại driver con là:
- Type Drivers dùng để định nghĩa loại network (như flat, vlan, vxlan, gre, geneve)
- Mechanism Drivers — quyết định cách thực thi
  - OVS (Open vSwitch): Truyền thống, dùng ovs-agent
  - OVN (Open Virtual Network):Hiện đại, thay thế OVS agent + L3 agent bằng control plane phân tán
  - LinuxBridge: Đơn giản, dùng Linux bridge thuần
  - SR-IOV: Bypass hypervisor, gắn VF trực tiếp vào VM
  - MacVTap:Tương tự SR-IOV nhưng dùng macvtap interface

3. L3 / Service Plugins
- L3 Agent (router): Router namespace truyền thống, dùng với OVS/LinuxBridge
- OVN L3: L3 tích hợp trong OVN, không cần L3 agent riêng
- FWaaS: Firewall-as-a-Service
- LBaaS / Octavia: Load Balancer — Octavia là backend hiện đại thay LBaaS v2
- VPNaaS: VPN-as-a-Service

3. IPAM Drivers
Quản lý địa chỉ IP:

- Internal (default) — IPAM nội bộ của Neutron
- Infoblox — tích hợp với Infoblox DDI

---
Sự khác nhau giữa 3 mechanism driver ovs/ ovn /và linux bridge

1. LinuxBridge — Đơn giản nhất
```
VM1 ──tap─┐
          ├── brqXXX (linux bridge) ── eth0 (physical)
VM2 ──tap─┘
```

Cách hoạt động:
- Mỗi network tạo 1 Linux bridge (brq<network-id>)
- VM gắn vào bridge qua tap interface
- VLAN dùng eth0.100 (vlan subinterface)
- VXLAN dùng vxlan-<vni> interface

Agent: neutron-linuxbridge-agent chạy trên mỗi compute node, lắng nghe RPC từ Neutron server, gọi brctl/ip link để cấu hình.
Ưu điểm: Dễ debug, không cần phần mềm thêm.
Nhược điểm: Không có flow-based forwarding, không hỗ trợ DVR, kém scale.

2. OVS (Open vSwitch) + Agent — Truyền thống
```
VM1 ──tap─┐
          ├── br-int ──patch──  br-tun ── VXLAN tunnel
VM2 ──tap─┘              └──patch── br-ex ── physical NIC
```
Cách hoạt động:
- 3 bridge chính: br-int (integration), br-tun (tunnel), br-ex (external)
- Forwarding dựa trên OpenFlow rules trong OVS
- Agent (neutron-openvswitch-agent) dịch Neutron intent → OpenFlow rules
- L3 Agent tạo network namespace (qrouter-xxx, qdhcp-xxx) cho router/DHCP

Agent stack:
```
Neutron Server
     │ RPC (AMQP)
     ▼
neutron-openvswitch-agent   ← cấu hình OVS flows
neutron-l3-agent            ← tạo qrouter namespace
neutron-dhcp-agent          ← tạo qdhcp namespace
```
Ưu điểm: Flow-based, hỗ trợ DVR, phổ biến, nhiều tài liệu.
Nhược điểm: Agent-based → mỗi thay đổi phải đi qua RPC → agent → OVS. Latency cao khi scale lớn. L3 agent là single point of failure nếu không dùng DVR/HA.

3. OVN (Open Virtual Network) — Hiện đại nhất
```
Neutron Server
     │ (ML2/OVN driver ghi trực tiếp)
     ▼
OVN Northbound DB  (logical: switches, routers, ACLs)
     │ ovn-northd
     ▼
OVN Southbound DB  (physical: flows, chassis info)
     │
     ├── ovn-controller (compute node 1) → OVS flows
     └── ovn-controller (compute node 2) → OVS flows
```
Cách hoạt động:
- Không có neutron agent riêng trên compute node
- ML2/OVN driver ghi logical network vào OVN Northbound DB
- ovn-northd compile logical → physical flows vào Southbound DB
- ovn-controller trên mỗi node tự đọc Southbound DB và lập trình OVS
- Router, DHCP, NAT xử lý trong OVS datapath — không cần namespace

Ưu điểm:

Distributed hoàn toàn — không có L3 agent, không có qrouter namespace
Floating IP, SNAT xử lý tại compute node (như DVR nhưng native)
Scale tốt hơn nhiều
DHCP trả lời trực tiếp từ OVN logical port (không cần qdhcp)

Nhược điểm: Khó debug hơn, cần hiểu 2 tầng DB (NB/SB).

----


OpenStack có hai thế giới cần nối với nhau:

Thế giới logic (trong Neutron): Khi bạn tạo một external network hay provider network, bạn khai báo kiểu như --provider-physical-network physnet1. Cái tên physnet1 này hoàn toàn là một label logic — Neutron không biết và không quan tâm nó tương ứng với cái gì trên host thật.

Trên mỗi compute/network node có thể có các các bridge OVS như br-ex, br-provider gắn với một NIC vật lý thật (ví dụ eth1, ens192...) để ra được mạng bên ngoài.

Khi một VM cần gửi traffic ra mạng external (ví dụ dùng floating IP), traffic đi từ br-int (bridge nội bộ của OVS) và cần thoát ra ngoài qua một NIC vật lý. Nhưng OVS agent trên host đó làm sao biết traffic cần ra mạng ngoài phải đi qua bridge nào, khi nó chỉ được cho biết tên logic là physnet1
OVS agent cần biết: physnet1 tương ứng với bridge nào trên node này? Nó nhìn vào bridge_mappings. bridge_mappings chính là bảng tra cứu để dịch từ tên logic sang bridge thật.

bridge_mappings = physnet1:br-ex
=> Dòng này nói rằng: "Trên node này, mạng logic tên physnet1 hãy đi ra qua bridge br-ex."

Giả sử bạn có hạ tầng như sau:
Trên Node A, NIC eth1 nối vào switch vật lý, đi ra internet. Bạn tạo bridge br-ex và add eth1 vào br-ex: `ovs-vsctl add-port br-ex eth1`
Trong file cấu hình OVS agent trên Node A:
```
[ovs]
bridge_mappings = physnet1:br-ex
```
Trong Neutron, bạn tạo external network:
```
openstack network create ext-net \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  --external
```
Lúc này chuỗi kết nối hoàn chỉnh là: `VM → tap → br-int → [patch port] → br-ex → eth1 → mạng vật lý`

Patch port ở giữa được tạo tự động nhờ bridge_mappings cho OVS biết physnet1 = br-ex.

Nếu không có bridge_mappings thì OVS agent không biết physnet1 map với bridge nào → không tạo patch port → traffic từ br-int không có đường ra br-ex → floating IP không hoạt động, provider network không thông.

br-ex (External Bridge) là bridge hướng ra ngoài, gắn với NIC vật lý (ví dụ eth1) để ra mạng external/internet. Nó chỉ phục vụ việc đẩy traffic ra khỏi node.

br-int kết nối với br-ex qua một cặp patch port — giống như một sợi dây ảo nối hai bridge lại:
```
br-int                          br-ex
┌──────────┐                   ┌──────────┐
│  VM1-tap │                   │   eth1   │ → mạng vật lý
│  VM2-tap │                   │          │
│          │                   │          │
│ patch-to-ex ──────────── patch-to-int   │
└──────────┘                   └──────────┘
```

Hai patch port này được OVS agent tự động tạo khi nó đọc bridge_mappings = physnet1:br-ex. Nếu không có bridge_mappings, hai bridge này tồn tại độc lập, không nối nhau, traffic không đi được.

Tách riêng để phân tách chức năng rõ ràng. br-int dùng chung cho mọi loại mạng (tenant network, provider network, VXLAN...). br-ex chỉ dành cho traffic ra ngoài. Nhờ tách ra, OVS có thể áp các flow rule khác nhau trên mỗi bridge — ví dụ trên br-int thì xử lý VLAN tagging nội bộ, security group, còn trên br-ex thì xử lý NAT, floating IP.

Ngoài ra, một node có thể có nhiều bridge external khác nhau (ví dụ br-ex cho mạng internet, br-provider cho mạng nội bộ công ty), mỗi cái map với một physical network khác nhau trong bridge_mappings: `bridge_mappings = physnet1:br-ex, physnet2:br-provider`

Các loại network trong OpenStack:
- Cần bridge_mappings: flat network và VLAN network (gọi chung là provider network). Đây là các mạng mà traffic phải đi ra NIC vật lý thật, nên OVS cần phải có bridge_mappings để OVS biết đi ra bridge nào . `VM → tap → br-int → patch port → br-ex/br-provider → NIC vật lý`
- Không cần bridge_mappings: VXLAN, GRE, Geneve (gọi chung là overlay/tunnel network). Đây là các tenant network thông thường (loại phổ biến nhất khi bạn tạo openstack network create my-net mà không chỉ định provider). Traffic được đóng gói (encapsulate) rồi gửi qua tunnel giữa các node, đi qua IP của chính các node — không cần bridge riêng, không cần map gì cả. OVS agent biết dùng IP của node (cấu hình trong local_ip) để thiết lập tunnel. Không liên quan gì đến bridge_mappings.
- External network thực chất cũng là một flat hoặc VLAN network, chỉ khác là được đánh dấu --external để Neutron biết đây là mạng dùng cho floating IP và router gateway. Nên nó cũng cần bridge_mappings — không phải vì nó "external" mà vì nó là flat/VLAN.

Tóm lại: config bridge_mappings dành cho bất kỳ network nào cần đi ra NIC vật lý — tức flat và VLAN. Overlay network (VXLAN/GRE/Geneve) thì không cần vì traffic đi qua tunnel, không cần bridge riêng.

Trường mapping này phải tự khai báo. OpenStack không tự biết NIC nào trên host dùng cho mạng nào.
Cấu hình ở đâu tùy vào backend bạn dùng:
- Với OVS agent (ML2/OVS): file /etc/neutron/plugins/ml2/openvswitch_agent.ini
- Với OVN: file /etc/neutron/plugins/ml2/ml2_conf.ini hoặc cấu hình trực tiếp trên ovn-controller qua ovs-vsctl: `ovs-vsctl set open . external-ids:ovn-bridge-mappings="physnet1:br-ex"`

Trước khi khai báo bridge_mappings, bạn phải tự tạo bridge và add NIC vật lý vào,OpenStack không làm bước này cho bạn.:
```
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex eth1
```

Ngoài ra còn phải khai báo phía Neutron server: Trong /etc/neutron/plugins/ml2/ml2_conf.ini trên controller node, bạn cần cho Neutron biết physical network nào cho phép loại mạng nào:
```
[ml2_type_flat]
flat_networks = physnet1

[ml2_type_vlan]
network_vlan_ranges = physnet2:100:200
```

Đây là khai báo phía server (Neutron biết có những physical network nào). Còn bridge_mappings là khai báo phía agent trên mỗi node (mỗi node biết physical network đó map với bridge nào của mình).

Tại sao không tự động: Vì mỗi node có thể có cấu hình mạng vật lý khác nhau — node A có eth1 cho external, node B có ens192, node C thậm chí không cần kết nối external. OpenStack không thể đoán được, nên bạn phải khai báo rõ trên từng node.


Bạn chỉ cần khai báo một lần cho mỗi physical network, không phải mỗi lần tạo network.

Phân biệt hai khái niệm:
- Physical network (physnet1, physnet2...): là label đại diện cho một hạ tầng mạng vật lý thật. Số lượng rất ít, thường chỉ 1-2 cái. Cái này bạn khai báo trong config.
- OpenStack network: là mạng logic bạn tạo bằng openstack network create. Số lượng có thể rất nhiều. Cái này bạn chỉ cần trỏ vào physical network đã khai báo sẵn.

Ví dụ thực tế
Bạn cấu hình một lần trên mỗi node: `bridge_mappings = physnet1:br-ex, physnet2:br-provider`

Sau đó bạn tạo bao nhiêu network cũng được, chỉ cần trỏ vào physical network đã có:
```
Tạo external network → dùng physnet1
openstack network create ext-net \
  --provider-physical-network physnet1 \
  --provider-network-type flat \
  --external

# Tạo thêm VLAN network cho team A → cũng dùng physnet2
openstack network create team-a-net \
  --provider-physical-network physnet2 \
  --provider-network-type vlan \
  --provider-segment 100

# Tạo thêm VLAN network cho team B → vẫn dùng physnet2
openstack network create team-b-net \
  --provider-physical-network physnet2 \
  --provider-network-type vlan \
  --provider-segment 101
```
Ba network khác nhau, nhưng không cần sửa config gì thêm. Vì physnet1 và physnet2 đã được map sẵn rồi.
Còn với overlay network (VXLAN/Geneve) thì càng đơn giản hơn — tạo bao nhiêu cũng được, không liên quan gì đến bridge_mappings

Tóm lại: Bạn chỉ cấu hình bridge_mappings khi thay đổi hạ tầng vật lý — thêm một đường mạng mới, thêm NIC mới. Còn tạo network logic trong OpenStack thì cứ tạo thoải mái, không cần động vào config.

---

Trong OpenStack Neutron, agent và plugin là hai khái niệm thường bị nhầm lẫn nhưng đóng vai trò rất khác nhau trong kiến trúc mạng.
Plugin
Plugin hoạt động ở phía server (neutron-server), chạy trong tiến trình của neutron-server. Nhiệm vụ chính của nó là:
- Nhận API request từ người dùng (tạo network, subnet, port, router…) và chuyển thành logic nghiệp vụ.
- Ghi trạng thái mong muốn (desired state) vào database.
- Quyết định cách mạng được triển khai — ví dụ dùng Linux Bridge, OVS, hay SR-IOV.

Kiến trúc hiện tại chia plugin thành hai tầng: core plugin (ML2 — Modular Layer 2) xử lý network/subnet/port, và service plugin xử lý router, firewall, VPN, LB… Bên trong ML2 lại có type driver (flat, VLAN, VXLAN, GRE) và mechanism driver (openvswitch, linuxbridge, OVN, SR-IOV) để tách biệt loại mạng và cách triển khai.

Agent
Agent là tiến trình chạy trên các node (compute, network, controller), nhận chỉ thị từ plugin qua RPC (RabbitMQ) rồi cấu hình thực tế trên hạ tầng. Một số agent phổ biến:
- neutron-openvswitch-agent / neutron-linuxbridge-agent — chạy trên compute và network node, cấu hình bridge, flow rules, VXLAN tunnel…
- neutron-l3-agent — tạo router namespace, iptables NAT, floating IP.
- neutron-dhcp-agent — chạy dnsmasq trong namespace để cấp DHCP.
- neutron-metadata-agent — proxy metadata service cho VM.

Luồng hoạt động minh hoạ: Khi gọi openstack port create, luồng đi như sau: neutron-server → ML2 plugin (chọn mechanism driver, ghi DB) → gửi RPC message → agent trên compute node nhận message → agent cấu hình OVS flow / Linux bridge / VIF binding cho port đó.

Trường hợp đặc biệt: OVN
Với ML2/OVN (mà Tuna đang tìm hiểu), mô hình hơi khác. OVN mechanism driver ghi trạng thái vào OVN Northbound DB thay vì gửi RPC. Sau đó ovn-northd dịch sang Southbound DB, và ovn-controller trên mỗi node (đóng vai trò tương đương agent) đọc Southbound DB rồi cấu hình OVS flow. Vì vậy với OVN, không cần neutron-l3-agent hay neutron-dhcp-agent truyền thống — OVN tự xử lý routing và DHCP bằng logical flow.

---

phân biệt project network và provider network: Đây là hai loại network cơ bản trong Neutron, khác nhau ở ai tạo, ai quản lý, và cách nó ánh xạ xuống hạ tầng vật lý.

Provider Network:
Provider network do admin tạo và ánh xạ trực tiếp xuống hạ tầng mạng vật lý bên dưới. Khi tạo, admin phải chỉ định rõ các tham số vật lý:
```
openstack network create --provider-network-type vlan \
  --provider-physical-network physnet1 \
  --provider-segment 100 \
  provider-net
```
Ở đây admin nói rõ: dùng VLAN 100, trên physical network physnet1 (tương ứng một bridge/interface thật trên host). VM gắn vào network này sẽ nằm cùng broadcast domain với hạ tầng vật lý bên ngoài — giống như cắm thẳng máy vào switch vật lý ở VLAN 100. Đặc điểm chính: VM có thể nhận IP từ DHCP server bên ngoài hoặc từ Neutron DHCP, và truy cập mạng bên ngoài trực tiếp mà không cần router ảo hay floating IP. Loại mạng thường là flat hoặc VLAN.


Project Network (Tenant Network)
Project network do người dùng thường tạo, không cần biết gì về hạ tầng bên dưới. Mạng này là overlay, hoàn toàn cô lập — hai project khác nhau có thể dùng cùng dải IP 10.0.0.0/24 mà không xung đột, vì traffic được đóng gói trong tunnel riêng.
VM trong project network không thể ra ngoài trừ khi gắn vào một router ảo mà router đó nối với external/provider network. Muốn truy cập từ ngoài vào thì cần floating IP.

So sánh nhanh
- Provider network chia sẻ hạ tầng vật lý (VLAN), nên số lượng giới hạn bởi VLAN ID (4094). Project network dùng overlay (VXLAN hỗ trợ ~16 triệu VNI), cô lập hoàn toàn giữa các tenant.
- Truy cập bên ngoài: Provider network có sẵn. Project network phải qua router + floating IP.
- Performance: Provider network thường nhanh hơn vì không có overhead đóng gói tunnel. Project network có thêm VXLAN/GRE header (~50 byte), và traffic phải đi qua network node nếu cần ra ngoài (trừ khi bật DVR).

Kiến trúc thực tế
Trong một deployment điển hình, cả hai loại cùng tồn tại. Provider network thường đóng vai trò external network — admin tạo nó, map vào VLAN trên uplink switch, rồi user tạo router gắn gateway vào đó. Project network là mạng nội bộ của từng tenant. Luồng traffic: `VM → project network (VXLAN tunnel) → router ảo trên network node → provider/external network (VLAN/flat) → ra ngoài.`

Với ML2/OVN mà Tuna đang làm việc, logic tương tự nhưng router ảo được xử lý phân tán bởi ovn-controller trên mỗi compute node (distributed routing mặc định), nên traffic east-west và cả north-south với floating IP không cần đi vòng qua network node riêng.



Không cần "tạo" physnet1 theo nghĩa chạy lệnh OpenStack nào cả — physnet1 là một label cấu hình, không phải một resource trong Neutron DB.
Việc Tuna cần làm là khai báo bridge_mappings trong file cấu hình của ovs/ovn trên mỗi node, để Neutron biết label physnet1 tương ứng với bridge/interface vật lý nào trên host.

trên ML2 config:
```
[ml2_type_flat]
flat_networks = physnet1

[ml2_type_vlan]
network_vlan_ranges = physnet1:100:200 #Dòng network_vlan_ranges nói Neutron được phép dùng VLAN 100–200 trên physnet1.
```

Tóm lại quy trình
- Bước 1: tạo OVS bridge br-ex, gắn NIC vật lý vào.
- Bước 2: khai báo bridge_mappings = physnet1:br-ex trong config.
- Bước 3: khai báo physnet1 trong flat_networks hoặc network_vlan_ranges của ML2.
- Bước 4: restart service. Sau đó khi --provider-physical-network physnet1 xuất hiện trong lệnh tạo network, Neutron biết chính xác phải đẩy traffic ra bridge nào trên host nào.


Trong ngữ cảnh OpenStack Neutron, physical network không phải là một thiết bị hay resource cụ thể — nó đơn giản là một cái tên (label) mà admin đặt để đại diện cho một đường kết nối vật lý ra bên ngoài.
Hãy hình dung thế này: trên mỗi host có thể có nhiều NIC vật lý, mỗi NIC nối vào một mạng vật lý khác nhau (ví dụ một NIC nối vào switch cho mạng public, một NIC nối vào switch cho mạng storage). Neutron cần một cách để đặt tên cho từng đường ra đó, và đó chính là physical network label.
Ví dụ thực tế: Tuna có một server với hai NIC — eth0 nối vào switch mạng public, eth1 nối vào switch mạng internal. Tuna tạo hai OVS bridge:

br-ex gắn eth0 → đặt tên là physnet-public
br-int-ext gắn eth1 → đặt tên là physnet-internal

Cấu hình sẽ là:
```
bridge_mappings = physnet-public:br-ex, physnet-internal:br-int-ext
```
Từ đó khi tạo provider network, Tuna chọn --provider-physical-network physnet-public nghĩa là traffic sẽ đi qua br-ex → eth0 → switch public. Còn --provider-physical-network physnet-internal thì đi qua br-int-ext → eth1 → switch internal.

Bản chất chuỗi mapping là: label → OVS bridge → NIC vật lý → switch/mạng thật. Physical network chỉ là mắt xích đầu tiên — cái tên mà Neutron dùng để trỏ tới cả chuỗi phía sau. Tuna muốn đặt tên gì cũng được (physnet1, public, datacenter-lan…), miễn là nhất quán giữa config ML2 và config agent trên tất cả các node.



Để mình giải thích từ góc nhìn hạ tầng vật lý trước.
Bắt đầu từ thực tế
Tuna có một datacenter, trong đó có các switch vật lý. Các switch này tạo ra các mạng vật lý — ví dụ:

Mạng công ty 192.168.1.0/24 trên VLAN 100
Mạng internet public trên VLAN 200
Mạng storage 10.0.0.0/24 trên VLAN 300

Đây là những mạng có thật, tồn tại trên switch, trên dây cáp, trước khi OpenStack được cài đặt. Máy chủ vật lý (compute node) cắm dây mạng vào các switch này.
Vấn đề cần giải quyết
Khi OpenStack chạy, VM bên trong cần kết nối ra các mạng vật lý đó. Nhưng Neutron không biết hạ tầng vật lý của Tuna trông như thế nào — nó không biết eth0 nối vào mạng nào, eth1 nối vào mạng nào.
Vì vậy admin phải "giới thiệu" cho Neutron biết, bằng cách đặt tên cho từng đường ra:
"physnet1"  →  nghĩa là đường ra mạng internet public
"physnet2"  →  nghĩa là đường ra mạng storage
Tên này hoàn toàn do Tuna tự đặt. Rồi trên mỗi host, Tuna nói cho Neutron biết tên đó ứng với bridge/NIC nào:
inibridge_mappings = physnet1:br-ex
Câu này nghĩa là: "cái mạng vật lý mà tôi gọi là physnet1, trên host này nó đi ra qua bridge br-ex".
Một phép so sánh đơn giản
Hãy tưởng tượng một toà nhà có nhiều cửa ra. Mỗi cửa dẫn ra một con đường khác nhau. "Physical network" giống như Tuna dán nhãn lên mỗi cửa: cửa này gọi là "đường-ra-internet", cửa kia gọi là "đường-ra-storage". Khi VM cần ra internet, Neutron nhìn nhãn, biết phải đi qua cửa nào.
Bản thân cái nhãn không tạo ra con đường — con đường (NIC, switch, cáp) đã có sẵn. Nhãn chỉ giúp Neutron biết đường nào là đường nào.



đường đi của traffic từ VM ra ngoài sẽ như thế nào
Mình sẽ giải thích cho cả hai trường hợp: provider network và project network.
Trường hợp 1: VM gắn trực tiếp vào provider network
Đường đi rất ngắn, giống như cắm máy thật vào switch:
VM → tap interface → br-int (OVS) → br-ex (OVS) → NIC vật lý (eth0) → switch vật lý → ra ngoài
Ở đây br-int và br-ex được nối với nhau qua patch port. Traffic từ VM đi ra ngoài mang đúng VLAN mà admin đã chỉ định khi tạo provider network. Không có tunnel, không có NAT, không có router ảo — VM nằm cùng broadcast domain với mạng vật lý bên ngoài.
Trường hợp 2: VM trong project network (VXLAN) muốn ra ngoài
Đây là trường hợp phức tạp hơn. VM không nằm trên mạng vật lý, mà nằm trong một overlay network cô lập. Để ra ngoài, traffic phải đi qua router ảo.
Với kiến trúc truyền thống (L3 agent trên network node)
Trên compute node:
VM → tap → br-int → đóng gói VXLAN → NIC vật lý → qua mạng vật lý đến network node
Trên network node:
NIC vật lý → tháo VXLAN → br-int → vào router namespace → SNAT (đổi IP private thành IP trên external subnet) → br-ex → NIC vật lý → switch → ra ngoài
Nếu VM có floating IP, traffic đến từ ngoài sẽ đi ngược: vào br-ex → router namespace → DNAT (đổi floating IP thành IP private) → đóng VXLAN → gửi về compute node → VM.
Điểm quan trọng: tất cả traffic north-south đều phải đi qua network node, nên network node dễ thành bottleneck.
Với DVR (Distributed Virtual Router)
Floating IP traffic được xử lý ngay trên compute node, không cần đi vòng qua network node:
VM → tap → br-int → router namespace trên chính compute node → DNAT/SNAT → br-ex → NIC vật lý → ra ngoài
Chỉ traffic SNAT (VM không có floating IP) mới phải đi về network node.
Với ML2/OVN
OVN làm distributed routing mặc định, tương tự DVR nhưng không dùng Linux namespace. ovn-controller trên mỗi compute node cài OVS flow để xử lý routing và NAT ngay tại chỗ. Đường đi:
VM → tap → br-int (OVS flow xử lý routing + NAT) → br-ex → NIC vật lý → ra ngoài
Không có router namespace, không có L3 agent — mọi thứ là OVS flow do ovn-controller lập trình từ Southbound DB.
Tóm lại sự khác biệt cốt lõi
Provider network: VM → bridge → NIC → ra ngoài (đường thẳng, không NAT).
Project network: VM → tunnel đến nơi có router → NAT → ra ngoài. "Nơi có router" là network node (truyền thống), hoặc chính compute node (DVR/OVN).
Tuna muốn mình đi sâu vào phần nào, ví dụ flow cụ thể trên OVS, hay cách floating IP hoạt động trong OVN?
