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

## Các lệnh làm việc với Openstack

### Compute (Nova)

Liệt kê instances
- openstack server list --all-projects
- openstack server list --project <project_id>

Tạo instance
- openstack server create --flavor m1.small --image ubuntu-22.04 --network <network-name> --key-name my-key --security-group default my-instance

Thao tác với instance
- openstack server show <server-id>
- openstack server start/stop/reboot <server-id>
- openstack server delete <server-id>
- openstack server suspend/resume <server-id>
- openstack server pause/unpause <server-id>

Console / log
- openstack console url show <server-id> --os-project-name <tên-project>
- openstack console log show <server-id>

Migration
- openstack server migrate <server-id>                    # cold
- openstack server migrate --live <host> <server-id>     # live

Resize
- openstack server resize --flavor m1.large <server-id>
- openstack server resize confirm <server-id>

Liệt kê hypervisors
- openstack hypervisor list
- openstack hypervisor show <hypervisor>
- openstack hypervisor stats show

### Network (Neutron)

Liệt kê và tạo network
- openstack network list
- openstack network create <name> [--external] [--provider-network-type flat]
- openstack network delete <name>

Liệt kê và tạo subnet
- openstack subnet list
- openstack subnet create --network <network> --subnet-range 192.168.1.0/24 --gateway 192.168.1.1 --dns-nameserver 8.8.8.8 my-subnet

Liệt kê và tạo Routers
- openstack router list
- openstack router create my-router
- openstack router set --external-gateway <ext-network> my-router
- openstack router add subnet my-router my-subnet
- openstack router remove subnet my-router my-subnet

Floating IPs
- openstack floating ip list
- openstack floating ip create <ext-network>
- openstack floating ip set --port <port-id> <floating-ip>
- openstack server add floating ip <server-id> <floating-ip>
- openstack server remove floating ip <server-id> <floating-ip>

Liệt kê và tạo Security Groups
- openstack security group list
- openstack security group create my-sg
- openstack security group rule create --protocol tcp --dst-port 22 --remote-ip 0.0.0.0/0 my-sg
- openstack security group rule list my-sg

Liệt kê và tạo Ports
- openstack port list --server <server-id>
- openstack port show <port-id>

### Storage (Cinder & Swift)

Liệt kê và tạo Volumes (Cinder)
- openstack volume list
- openstack volume create --size 20 --type <type> my-volume
- openstack volume show <volume-id>
- openstack volume delete <volume-id>

Attach/Detach
- openstack server add volume <server-id> <volume-id>
- openstack server remove volume <server-id> <volume-id>

Snapshots
- openstack volume snapshot create --volume <vol-id> my-snap
- openstack volume snapshot list
- openstack volume snapshot delete <snap-id>

Volume types
- openstack volume type list

Object Storage (Swift)
- openstack container list
- openstack container create my-bucket
- openstack object list my-bucket
- openstack object upload my-bucket /path/to/file
- openstack object save my-bucket <object> --file /tmp/out

### Images (Glance)

Liệt kê images
- openstack image list
- openstack image show <image-id>

Upload image
- openstack image create --file ubuntu-22.04.img --disk-format qcow2 --container-format bare --public ubuntu-22.04

Download & delete
- openstack image save --file /tmp/out.qcow2 <image-id>
- openstack image delete <image-id>

Chia sẻ image giữa projects
- openstack image add project <image-id> <project-id>
- openstack image set --shared <image-id>

### Identity (Keystone)
Projects
- openstack project list
- openstack project create <name> --domain Default
- openstack project delete <name>

Users
- openstack user list
- openstack user create --password <pass> --email x@x.com <name>
- openstack user delete <user>

Roles
- openstack role list
- openstack role add --user <user> --project <project> member
- openstack role assignment list --project <project>

### Theo dõi & Debug
Quota
- openstack quota show <project>
- openstack quota set --instances 50 --cores 200 --ram 204800 <project>

Usage
- openstack usage list
- openstack usage show --start 2025-01-01 --end 2025-12-31

Services & Agents
- openstack service list
- openstack compute service list
- openstack network agent list
- openstack volume service list

Kiểm tra logs (trên node)
- journalctl -u nova-api -f
- journalctl -u neutron-server -f
- journalctl -u cinder-api -f
- tail -f /var/log/nova/nova-api.log
- tail -f /var/log/neutron/neutron-server.log

Endpoints
- openstack endpoint list
- openstack endpoint show <endpoint-id>

### Tips hữu ích
Filter
- openstack server list --status ERROR
- openstack server list --host <compute-node>
- openstack volume list --status error

Bulk delete instances lỗi
- openstack server list --status ERROR -f value -c ID | xargs -I{} openstack server delete {}

---

### Kiến trúc node trong OpenStack
OpenStack chia các chức năng ra nhiều node để tách biệt tải và trách nhiệm:

#### Controller Node
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


#### Compute Node
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

#### Network Node
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

#### Storage Node (tuỳ chọn)
- Cung cấp block storage và object storage:
  - Cinder — block storage (giống EBS của AWS)
  - Swift / Ceph — object storage (giống S3)
- Các process chạy trên storage node
  - cinder-volume     → Tạo/attach block volume
  - ceph-osd          → Object storage daemon (nếu dùng Ceph)
  - ceph-mon          → Ceph monitor

---

- Network trong Openstack là lớp hạ tầng mạng ảo, còn Subnet là phân vùng IP cụ thể
- Một network có thể có nhiều subnet
- OpenStack yêu cầu chỉ định network khi tạo instance, nhưng thực ra instance nhận IP từ subnet bên trong network đó:
- Subnet AWS thực sự là mạng con — tức là dải IP của subnet phải nằm trong dải IP của VPC. VPC là /16, subnet phải là /24 hoặc nhỏ hơn bên trong đó. Còn Network trong OpenStack không có dải IP — nó chỉ là lớp kết nối (Layer 2). Dải IP hoàn toàn thuộc về Subnet, không phải chia ra từ Network. Nên có thể tạo network "internal-network" và Subnet A có IP 10.0.0.0/24 và Subnet B có IP  192.168.50.0/24  (hoàn toàn khác dải, vẫn hợp lệ)

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

## Openstack Networking

<img width="701" height="590" alt="image" src="https://github.com/user-attachments/assets/b7160aff-001d-4047-9e1e-20ff47ce3d0b" />

Giải thích các khái niệm
- ML2 Plugin (Modular Layer 2): là core plugin duy nhất của Neutron hiện tại, thay thế các monolithic plugin cũ. "Modular" vì nó cho phép kết hợp linh hoạt các driver. ML2 gồm 3 loại driver hoạt động độc lập với nhau:
  - Type Drivers: định nghĩa loại network vật lý được dùng để vận chuyển traffic. flat là mạng không có VLAN tag, vlan dùng 802.1Q, vxlan/geneve/gre là các overlay tunnel protocol. Setup của bạn với OVN thường dùng geneve.
  - Mechanism Drivers — định nghĩa cách implement network đó trên thực tế (ai lập trình switch ảo, ai xử lý routing). Đây chính là thứ người ta hay gọi không chính thức là "backend" hoặc "ML2 driver". Ba tên gọi này đều chỉ cùng một thứ. Mechanism Driver được cấu hình ở dòng `mechanism_drivers =` trong ml2_conf.ini.
  - Extension Drivers — các tính năng tùy chọn gắn thêm vào port/network như QoS, port security, DNS. Không liên quan đến data plane.

### Mechanism Drivers (backend/ML2 driver)
- Chức năng: Biến "khai báo mạng" thành "mạng thật sự hoạt động". Khi chạy các lệnh như `openstack network create my-net`, `openstack router create my-router`, `openstack server create --network my-net my-vm`, Neutron chỉ lưu thông tin vào databasen nhưng chưa thực sự cấu hình ở tầng hệ điều hành và phần cứng.. Backend networking là thứ biến các bản ghi trong DB đó thành hạ tầng mạng thật sự.
- Cụ thể công việc của backend networking khi bạn tạo một VM và gán vào network:
    - Trên compute node: tạo virtual port, kết nối VM vào virtual switch (OVS bridge hoặc Linux bridge), áp dụng security group rules (iptables hoặc OVS flows) để filter traffic của VM đó.
    - Trên network node: tạo router ảo (qrouter namespace với L3 Agent, hoặc logical router trong OVN), cấu hình routing giữa các subnet, setup NAT cho floating IP, chạy DHCP server để VM lấy được IP.
    - Giữa các node: tạo tunnel (Geneve/VXLAN) để traffic của VM trên compute01 có thể đi sang VM trên compute02, dù hai VM đang ở hai máy vật lý khác nhau.
- Có các type phổ biến là OVN, openvswitch (mặc định của Neutron) và SR-IOV
- Hầu hết các installer hiện đại (Kolla-AnsibleOVN, OpenStack-AnsibleOVN, Canonical MicroStack,...) đều mặc định dùng OVN vì đó là hướng phát triển chính của OpenStack từ khoảng bản Yoga (2022) trở đi

#### 1. Neutron truyền thống (L3 Agent)
<img width="788" height="530" alt="image" src="https://github.com/user-attachments/assets/e6fe716b-3d02-4c76-93ad-f9778025bedb" />
- Neutron truyền thống dùng các agent chạy ở userspace trên Network node. Khi bạn tạo 1 virtual router, L3 agent sẽ tạo ra 1 Linux network namespace (qrouter-xxx) với iptables rules và ip route riêng. 10 router = 10 namespace = 10 iptables chain. Traffic đi qua userspace nên chậm hơn và tốn CPU.
- Mỗi virtual router tạo ra một qrouter-* namespace độc lập trên Network node, mỗi network tạo ra một qdhcp-* namespace. 100 router = 100 namespace.
- Routing xử lý ở userspace (iptables), tốn tài nguyên và khó scale.

#### 2. OVN (Open Virtual Network)

<img width="800" height="523" alt="image" src="https://github.com/user-attachments/assets/f8c0ca68-9c2a-4e8c-83e1-223b84f509b4" />

- ovn-northd trên Controller đọc logical config từ NB DB (Northbound Database), dịch nó thành OVS flow rules, rồi ghi vào SB DB (Southbound Database).
- Sau đó ovn-controller trên mỗi node (network01, compute01) đọc SB DB và lập trình trực tiếp vào OVS kernel flow tables.
- Không có namespace, không có iptables — routing xảy ra trong kernel thông qua OVS, nhanh hơn nhiều.

#### 3. SR-IOV
- Dùng khi cần performance cực cao (telco, HPC). VM được cấp virtual function (VF) của NIC vật lý, bypass hoàn toàn OVS/kernel networking.
- Latency rất thấp nhưng mất tính flexibility (không dùng được floating IP, security group giới hạn).

---

<img width="801" height="485" alt="image" src="https://github.com/user-attachments/assets/7fb7196d-4f15-4d01-b385-13de01f15b1f" />

Linux Bridge agent — agent chạy trên mỗi compute node, tự tạo bridge bằng lệnh ip link add brq-xxx type bridge khi VM đầu tiên thuộc network đó được lên lịch chạy trên node. Nếu node chưa có VM nào của Network A thì chưa có bridge nào cho Network A. Có bao nhiêu network có VM trên node → có bấy nhiêu bridge.
OVN (setup của bạn) — ovn-controller tạo br-int ngay khi service khởi động, không cần đợi VM nào. Toàn bộ vòng đời của VM chỉ liên quan đến việc thêm/xóa port trên br-int và cập nhật OVS flow rules — không bao giờ tạo thêm bridge mới.
br-ex là ngoại lệ — không do agent tạo động, mà được tạo sẵn lúc cài đặt OpenStack (Kolla, manual...) và gán physical interface vào đó một lần duy nhất. Đó là lý do bạn thấy enp1s0 nằm trong br-ex khi chạy ovs-vsctl show.

Trả lời thẳng câu hỏi
Mỗi node không chỉ có đúng 1 bridge. Số lượng phụ thuộc backend:
BackendCompute nodeNetwork nodeLinux Bridgebrq-xxx × số network có VMbrq-xxx + vxlan interfaceOVS Agentbr-int + br-tun (cố định)br-int + br-tun + br-exOVN (bạn)br-int duy nhấtbr-int + br-ex
Với OVN trên cluster của bạn, chạy ovs-vsctl show trên compute01 sẽ chỉ thấy br-int — mọi VM tap interface đều kết nối vào đó, OVS flow tables xử lý phần còn lại trong kernel.
