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

Project trong OpenStack
Project là gì?
Project (trước đây gọi là Tenant) là đơn vị tổ chức và phân tách tài nguyên trong OpenStack. Mọi resource (VM, network, volume...) đều thuộc về một project.

Hiểu đơn giản: Project giống như một "không gian làm việc riêng" — các team/khách hàng khác nhau có project riêng, tài nguyên không ảnh hưởng lẫn nhau.


Project quản lý những gì?
Project "team-backend"
    ├── Compute (Nova)
    │       ├── VM: web-server-01
    │       └── VM: api-server-01
    ├── Network (Neutron)
    │       ├── Network: private-net
    │       └── Router: main-router
    ├── Storage (Cinder)
    │       └── Volume: data-disk-100GB
    ├── Image (Glance)
    │       └── Image: ubuntu-22.04-custom
    └── Quota
            ├── Max 10 VMs
            ├── Max 50 vCPUs
            └── Max 100GB RAM

Mối quan hệ Project - User - Role
         User "alice"
              │
    ┌─────────┼──────────┐
    │         │          │
    ▼         ▼          ▼
Project A  Project B  Project C
(role:     (role:     (role:
 admin)     member)    reader)

Một user có thể thuộc nhiều project với role khác nhau
Một project có thể có nhiều user
Role quyết định user được làm gì trong project đó


Các Role phổ biến
RoleQuyền hạnadminToàn quyền trong projectmemberTạo/xóa resource thông thườngreaderChỉ xem, không thay đổi

Project vs Domain
Domain "newwave"
    ├── Project "production"
    │       ├── User: alice (admin)
    │       └── User: bob (member)
    ├── Project "staging"
    │       └── User: alice (member)
    └── Project "dev"
            └── User: charlie (admin)

Domain là cấp cao hơn, nhóm các project lại
Mặc định Kolla dùng domain "Default"


Thao tác cơ bản với Project
bash# Liệt kê tất cả project
openstack project list

# Tạo project mới
openstack project create --domain Default --description "Backend Team" team-backend

# Xem chi tiết project
openstack project show team-backend

# Thêm user vào project với role
openstack role add --project team-backend --user alice member

# Xem quota của project
openstack quota show team-backend

# Xóa project
openstack project delete team-backend

Tại sao Project quan trọng?
Tính năngÝ nghĩaIsolationTài nguyên của project A không nhìn thấy từ project BQuotaGiới hạn tài nguyên từng project, tránh một team dùng quá nhiềuBillingTính tiền theo project (cho môi trường cloud thương mại)SecurityPhân quyền rõ ràng theo team/môi trường

Ví dụ thực tế tại New Wave
Có thể tổ chức như sau:
Domain: newwave
    ├── Project: production    → môi trường prod
    ├── Project: staging       → môi trường staging  
    ├── Project: dev           → môi trường dev
    └── Project: admin         → quản trị hệ thống
Mỗi project có quota riêng, team riêng, network riêng — hoàn toàn độc lập với nhau.
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
- openstack console url show <server-id>
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
