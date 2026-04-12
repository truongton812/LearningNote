# Container Network Interface (CNI)

## 1. Định nghĩa

- CNI là interface giữa container runtime (như containerd, podman,...) và các network plugin, giúp chúng hoạt động với nhau mà không cần phụ thuộc vào nhau.
- Các thành phần của CNI
  - Specification: Định nghĩa cách một plugin nhận input và trả output — thông qua biến môi trường và stdin/stdout theo định dạng JSON.
  - Plugin: Là các file thực thi (executables) thực hiện việc cấu hình mạng.
  - Library (libcni): Thư viện Go giúp tích hợp CNI vào container runtime dễ dàng hơn.

### 1.1. CNI specification

- CNI specification định nghĩa ra:
  - Định dạng chuẩn (thường là file JSON/YAML) để admin định nghĩa cấu hình mạng
  - Giao thức chuẩn (gọi qua command-line như plugin_linux ADD/DEL) để runtime (containerd, CRI-O, Docker) gửi request tới plugin CNI.
  - Quy trình để thực thi các plugins dựa trên network configuration ở trên
  - Quy trình để plugin delegate (gọi nhúng) plugin con để xử lý phần việc, ví dụ: host-local delegate cho IPAM
  - Định dạng JSON chuẩn cho plugin trả kết quả cho runtime

- Ví dụ cluster dùng CNI plugin Calico và container runtime là containerd, mỗi khi chạy lệnh `kubectl run nginx --image=nginx`, Kubernetes sẽ yêu cầu containerd khởi tạo container. containerd sẽ liên hệ với CNI plugin để attach mạng cho container đó. Giao tiếp giữa containerd và CNI plugin phải tuân theo chuẩn do CNI Specification đặt ra:
  - Admin cần định nghĩa cấu hình mạng trong thư mục `/etc/cni/net.d/10-calico.conflist` theo chuẩn CNI specification (mạng tên gì, dùng plugin gì, dải IP cho pod là gì,...)
  - Containerd đọc file config CNI và gọi file plugin CNI binary Calico theo chuẩn, ví dụ: `/opt/cni/bin/calico ADD`
  - Plugin Calico thực thi theo quy chuẩn. Ví dụ: Tạo veth pair trong container -> Gán IP từ dải mới cho container -> Cập nhật iptables/routes nếu cần.
  - Quy trình để CNI plugin Calico có thể gọi các CNI plugins khác xử lý công việc
  - Plugin trả kết quả về cho containerd theo định dạng JSON chuẩn. Containerd sẽ dựa vào kết quả này để gắn IP đúng cho interface trong container, thiết lập default gateway, routes....

### 1.2. CNI Plugin và CNI agent

CNI trong K8s gồm 2 phần tách biệt là CNI plugin và CNI agent

#### 1.2.1. CNI plugin
- Là các file binary (ví dụ /opt/cni/bin/flannel, /opt/cni/bin/calico) được kubelet gọi trực tiếp khi tạo/xóa pod. Chúng chạy như một process tạm thời (fork từ kubelet), KHÔNG chạy dưới dạng pod.
- Về cơ bản, có thể tự viết CNI plugin bằng bất kỳ ngôn ngữ nào (Go, Bash, Python, Rust…) miễn là nó ra một binary chạy đúng giao thức CNI. Ví dụ một repo [custom-cni](https://github.com/ronak-agarwal/custom-cni) mô tả cách viết một CNI plugin đơn giản cho k8s .

#### 1.2.2. CNI agent
- Là control plane của CNI (quản lý routing, IPAM, policy...) thường chạy dưới dạng DaemonSet Pod làm các nhiệm vụ:
  - Copy binary vào /opt/cni/bin/
  - Ghi config vào /etc/cni/net.d/
  - Quản lý routes, iptables/eBPF rules
  - Sync trạng thái giữa các node
  - **Không** tham gia trực tiếp vào quá trình tạo network cho từng pod.
- Khác với CNI binary chỉ chạy 1 lần khi pod create/delete rồi exit ngay sau khi xong việc, CNI daemon cần phải chạy liên tục để:
  - Duy trì routing table: Node A biết pod 10.0.1.5 nằm ở Node B. Nếu Node B reboot → IP thay đổi → Daemon phải update route ngay lập tức → Nếu daemon không chạy: traffic bị blackhole
  - BGP peering (nếu dùng Calico/Cilium): Daemon giữ BGP session liên tục với các node khác. Session drop → toàn bộ cross-node traffic mất
  - Watch K8s events: khi NetworkPolicy thay đổi → Daemon nhận event → Update iptables/eBPF rules ngay lập tức. Nếu daemon chết policy cũ vẫn còn nhưng policy mới không được apply
  - Health check & sync định kỳ: So sánh desired state vs actual state trên node, phát hiện drift (ai đó chạy iptables -F chẳng hạn) và reconcile lại
- CNI agent pod thường dùng `hostNetwork: true` và mount thư mục `/opt/cni/bin` và `/etc/cni/net.d` từ host do nó cần can thiệp vào network namespace của host.
- CNI agent có 2 phần là Control Plane và Data Plane. 
  - Control Plane giao tiếp với API server của Kubernetes, nhận thông tin về pod mới được tạo/xóa, tính toán cấu hình mạng cần thiết (IP allocation, network policy, routing rules), rồi đẩy xuống cho Data Plane thực thi.
  - Data Plane chịu trách nhiệm xử lý packet. Nó nhận chỉ thị từ Control Plane và lập trình vào kernel (qua eBPF, iptables, OVS, hoặc Linux routing table). Khi một packet đến, Data Plane xử lý hoàn toàn trong kernel space
- CNI agent có 2 kiến trúc: Centralized control plane và Distributed control plane

##### 1.2.2.1. Distributed control plane

  <img width="661" height="513" alt="image" src="https://github.com/user-attachments/assets/22413f66-c248-488c-8433-aab4058dd63c" />

- Là mô hình control plane và data plane cùng nằm trong một CNI daemon. Mỗi node có 1 CNI daemon, không có node nào là "master", tất cả đều ngang hàng. Mỗi node tự watch K8s API server để lấy NetworkPolicy, Pod events.
- Trong mô hình này thì số session BGP là O(N²) (full mesh). VD có 10 nodes thì sinh ra 45 sessions, 100 nodes thì có 4950 sessions. Đây là lý do Calico khuyến nghị dùng BGP Route Reflector (tức là một phần của mô hình centralized) khi cluster vượt quá ~50 nodes — để giảm từ O(N²) xuống còn O(N) sessions.

##### 1.2.2.2. Centralized control plane
  <img width="671" height="561" alt="image" src="https://github.com/user-attachments/assets/a6e26f2a-b49a-4c0a-bfcc-3206e7c87139" />

- Là mô hình control plane và data plane phân tách ra. Control Plane được deploy như một deployment riêng (thường là DaemonSet trên master nodes). Data Plane vẫn là DaemonSet chạy trên mọi worker node. Hai bên giao tiếp qua gRPC/REST.
- CNI control cluster watch API server để lấy NetworkPolicy, Pod events, CRD changes. Ba thành phần core là IPAM để cấp phát IP pool cho từng cluster, BGP reflector làm peer trung tâm của mạng BGP, Policy controller để dịch NetworkPolicy từ K8s CRD thành rule cụ thể rồi push xuống agent
- CNI agent ở mỗi node chỉ lo data plane: setup veth, apply eBPF/iptables rules, giữ BGP session với reflector ở trên. Không tự tính toán gì. Cross-cluster traffic vẫn dùng BGP, nhưng route decision đã được tính sẵn bởi BGP reflector ở control cluster, không phải bởi từng agent.
- Đây là kiến trúc áp dụng trong Cilium Enterprise và Calico Enterprise đang đi với mô hình  + distributed data plane.

## 2. Gán mạng cho Pod
- Trong kubernetes, khi 1 Pod được tạo ra, nó chỉ có một network namespace rỗng, không thể giao tiếp với các Pod khác. CNI plugin sẽ tạo veth, gán IP, rồi đưa một đầu veth vào trong namespace của pod. Veth pair là cầu nối cho phép packet đi xuyên qua ranh giới namespace mà không cần routing hay NAT. Packet xuất hiện ở 1 đầu của pair thì luôn luôn xuất hiện ở đầu còn lại
- Mặc định CNI chỉ attach 1 network vào interface chính của pod (eth0). Runtime đọc file .conf có số thứ tự thấp nhất trong thư mục `/etc/cni/net.d/` và dùng nó. Mọi pod đều vào cùng một flat network.
- Nếu muốn nhiều interface / nhiều mạng trên cùng một pod, cần dùng Multus CNI đứng trước các CNI khác. Multus CNI là một `meta CNI plugin` cho phép gắn nhiều interfaces thuộc nhiều mạng khác nhau cho một pod

### 2.1 Cách Multus hoạt động
- Multus sẽ đọc các file CNI config (ví dụ: dbnet, default, management) và expose cho Kubernetes dưới dạng `NetworkAttachmentDefinition` resource. VD:
  ```
  apiVersion: k8s.cni.cncf.io/v1
  kind: NetworkAttachmentDefinition
  metadata:
    name: dbnet
    namespace: default
  spec:
    config: '{ #config ở đây là phiên bản thu gọn của file CNI dbnet bạn có, chuyển thành JSON trong chuỗi.
      "cniVersion": "1.1.0",
      "type": "bridge",
      "bridge": "cni0",
      "ipam": {
        "type": "host-local",
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1"
      }
    }'
  ```

- Attach dbnet network vào pod thông qua annotation

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: db-pod
    namespace: default
    annotations:
      k8s.v1.cni.cncf.io/networks: dbnet
  ```

→ Khi pod chạy: Interface chính (eth0) do CNI mặc định (Calico/Cilium…) tạo. Interface thứ hai (ví dụ: net1) sẽ thuộc mạng dbnet (nếu dùng Multus kết hợp CNI mặc định).

- Nếu muốn pod chỉ dùng mạng dbnet hoàn toàn, thay đổi cấu hình Multus trên node để đặt dbnet làm interface chính.

## 3. Linux bridge 
- Là một virtual network switch được implement trong Linux kernel, có chức năng giống như một switch vật lý Layer 2 nhưng hoàn toàn bằng software. Thay vì port vật lý, bridge có các network interface gắn vào làm port. Đối với host Linux bridge xuất hiện là một interface trong host network namespace, giống như bất kỳ interface nào khác (eth0, lo...). Có thể show bằng lệnh `ip link show` → Sẽ thấy cni0/docker0 nằm chung với eth0, lo, flannel.1...
- Docker và K8s sử dụng Linux bridge để implement network vì nó có sẵn trong kernel, không cần cài thêm gì. Trong Docker bridge thường có tên là `docker0` do Docker daemon tạo ra. Còn trong Kubernetes thì bridge do CNI tạo và có tên là `cni0`
- Cách hoạt động: Bridge duy trì một Forwarding Database (FDB) —  là bảng ánh xạ MAC address → port, giống MAC table của switch thật. Khi container A gửi packet đến container B trong cùng mạng
  - Packet đi qua veth pair vào bridge
  - Bridge tra FDB theo MAC đích
  - Forward ra đúng port → vào veth pair của container B. Toàn bộ quá trình xảy ra trong kernel memory, không ra NIC vật lý
- Các lệnh làm việc với Bridge:
  - Tạo bridge: `ip link add name br0 type bridge && ip link set br0 up`
  - Liệt kê các bridge hiện có: `ip link show type bridge`
  - Xem FDB của bridge: `bridge fdb show br docker0`
  - Đặt IP cho bridge (để host giao tiếp được): `ip addr add 192.168.1.1/24 dev br0`
  - Gắn interface vào bridge (làm "port"): `ip link set veth0 master br0`
  - Xem port nào đang gắn vào bridge: `ip link show master cni0`
    
### 3.1. Linux bridge trong K8s
- Các CNI plugin phổ biến tạo bridge cni0 trong host network namespace.
- Khi 1 pod trong cụm k8s được tạo ra thì interface của nó sẽ được gắn vào bridge. Show trên host sẽ thấy các veth
- Lưu ý có các plugin không dùng bridge. VD Calico gắn veth trực tiếp vào routing table của host (cần kernel xử lý L3 routing cho từng pod)

### 3.2 OVS
<img width="1121" height="627" alt="image" src="https://github.com/user-attachments/assets/882c2207-21f8-4749-acd1-53c4e73574ea" />

- OVS là virtual switch thay thế cho Linux bridge với nhiều tính năng hơn
- Linux bridge chỉ có kernel module. Còn OVS có 3 thành phần:
  - ovsdb-server: lưu config (bridges, ports, flows)
  - ovs-vswitchd: xử lý control plane, cài flow vào kernel
  - openvswitch.ko: fast path, forward packet theo flow table
- Điểm khác biệt chính giữa OVS và Linux bridge là OVS làm tunnel native, còn Linux bridge cần thêm flannel.1 VTEP bên ngoài để làm VXLAN:
  - Linux bridge cần interface riêng cho tunnel: cni0 (bridge) → flannel.1 (VTEP) → eth0
  - OVS tự làm tunnel ngay trong switch

## 4. Giao tiếp giữa các Pod (container) trong Kubernetes
Có 2 loại là intra-node network và inter-node network

### 4.1. Intra-node Container Networking 
<img width="661" height="511" alt="image" src="https://github.com/user-attachments/assets/3b2ae263-9870-48d6-818e-8ddb4b601569" />

- Là giao tiếp giữa các container trong cùng một node. Lúc này Linux kernel sẽ xử lý toàn bộ traffic, không cần đi qua NIC vật lý.
- Linux Bridge thực hiện MAC-based forwarding (L2), hoặc kernel routing nếu khác subnet (L3). Ưu điểm là latency rất thấp vì chỉ là memory copy giữa các network namespace. Trong Kubernetes Bridge thường là cni0, do CNI plugin tạo ra.

### 4.2. Inter-node Container Networking
<img width="699" height="597" alt="image" src="https://github.com/user-attachments/assets/0c76f6db-a689-452d-8bd5-67ab2b97105c" />

Là trường hợp các container giao tiếp khác node, phức tạp hơn vì packet phải đi qua mạng vật lý (underlay). Có nhiều cách để giải quyết bài toán này:
- Overlay Network (VXLAN / Geneve): Dùng bởi Flannel VXLAN, Calico VXLAN, Cilium. Pod IP packet được đóng gói (encapsulate) vào UDP frame trước khi gửi qua mạng vật lý. Ưu điểm: Không yêu cầu cấu hình router/switch phức tạp. Nhược điểm: Overhead encap/decap, MTU phải giảm (~50 bytes cho VXLAN header).
- Native Routing (BGP / Direct Route): Dùng bởi Calico BGP, Cilium native routing, Flannel host-gw. Ưu điểm: Performance tốt nhất, không MTU overhead, dễ debug. Nhược điểm: Yêu cầu router/switch hỗ trợ, hoặc tất cả node phải cùng L2 segment (với host-gw).
- eBPF Datapath: Dùng bởi Cilium


## 5. Kubernetes trong Openstack

### 5.1. Normal CNI plugin
<img width="695" height="665" alt="image" src="https://github.com/user-attachments/assets/c4950e77-2788-4784-a1f5-6b8b8f0c3581" />

- Trong openstack, Pod là thành phần nằm bên trong VM. VM do OpenStack quản lý, còn Pod chạy bên trong VM đó — VM đóng vai trò là Kubernetes node. Do đó sẽ có hai lớp networking hoàn toàn tách biệt
  - Neutron quản lý network giữa các VM → VM thấy nhau qua 192.168.x.x
  - CNI plugin quản lý network giữa các Pod → Pod thấy nhau qua 10.244.x.x
  - Luồng traffic: `Pod A/B → veth pair → cni0 bridge (L2 switching trong VM) → kernel routing → eth0 của VM (192.168.1.10, Neutron network) → tap interface → OVS bridge trên hypervisor`
  - Khi Pod A (VM1) nói chuyện với Pod B (VM2): Pod traffic phải đi xuyên qua VM network của OpenStack — đây là lý do khi dùng Overlay (VXLAN) trong k8s trên OpenStack sẽ bị double encapsulation: VXLAN của CNI nằm trong VXLAN của Neutron.
- Trong sơ đồ trên, cni0 đóng vai trò là gateway của pod (cni0 cũng có IP riêng). cni0 nối với eth0 của VM thông qua kernal routing table. eth0 của VM đóng vai trò uplink cho toàn bộ pod network — mọi traffic ra ngoài VM đều phải đi qua đây, sau đó mới xuống OVS bridge của OpenStack.

### 5.2. OVS-CNI plugin
<img width="686" height="428" alt="image" src="https://github.com/user-attachments/assets/befb0c76-9dbc-48ab-ba08-d5bbc3f889f1" />

- OVS-CNI là một CNI plugin cho phép Kubernetes pod gắn network interface trực tiếp vào Open vSwitch (OVS) bridge trên node, thay vì dùng Linux bridge
Kiến trúc cơ bản
- Có thể tận dụng OVS-CNI plugin để kết nối pod network với Neutron OVS backend
- Lưu ý OVS-CNI không tự tạo OVS bridge. Nó expect bridge đã tồn tại sẵn trên **K8s node**. Không thể tận dụng OVS bridge của Openstack hypervisor bên dưới do K8s node chỉ thấy eth0 - một virtual NIC bình thường. Nó không có visibility vào lớp OpenStack hypervisor. Samsung đưa ra giải pháp là custom CNI Hyperplane - gọi Neutron API để tạo port

K8s Pod ──► Kuryr CNI ──► Neutron API ──► OVS (Neutron) ──► Network

Kuryr không dùng OVS trực tiếp, mà gọi Neutron API để tạo port
Neutron tự xử lý phía OVS bên dưới
Pod IP lấy từ Neutron subnet → cùng network plane với VM
Không cần overlay (VXLAN) trong K8s vì Neutron đã lo

Kuryr không dùng OVS trực tiếp, mà gọi Neutron API để tạo port
    •    Neutron tự xử lý phía OVS bên dưới
    •    Pod IP lấy từ Neutron subnet → cùng network plane với VMApr 11OK để mình giải thích từng lớp một, bắt đầu từ nền tảng.

1. Neutron hoạt động như thế nào (không có K8s)
Khi bạn tạo một VM trên OpenStack:
Nova yêu cầu Neutron: "Tạo cho tôi một network interface"
         ↓
Neutron tạo một "Port" (logical) trong database
         ↓
Neutron agent trên hypervisor node nhận event
         ↓
Agent gọi ovs-vsctl để tạo port thật trên OVS bridge
         ↓
VM được cắm vào OVS bridge qua tap interface
Neutron trừu tượng hóa OVS — bạn chỉ cần gọi API, còn việc tạo OVS port thật là việc của Neutron agent.

2. Bình thường K8s CNI làm gì
Khi pod được tạo, CNI plugin (Flannel/Calico) làm:
Kubelet gọi CNI plugin
         ↓
CNI tự tạo veth pair
CNI tự xin IP từ IPAM của nó (etcd/host-local)
CNI tự tạo route, iptables rules
         ↓
Hoàn toàn độc lập với OpenStack
K8s không biết Neutron tồn tại. Pod IP là một dải riêng (ví dụ 10.244.x.x), tách biệt hoàn toàn với Neutron subnet.

3. Kuryr làm khác — nó "nói chuyện" với Neutron
Đây là điểm mấu chốt:
Kubelet gọi Kuryr CNI plugin
         ↓
Kuryr KHÔNG tự tạo network interface
Kuryr gọi Neutron API: "Tạo cho tôi một Port trong subnet X"
         ↓
Neutron tạo Port, trả về: IP = 192.168.1.50, MAC = aa:bb:cc:...
         ↓
Neutron agent trên node tạo OVS port thật (giống như tạo cho VM)
         ↓
Kuryr gắn OVS port đó vào network namespace của pod
Pod bây giờ có IP 192.168.1.50 — chính là IP từ Neutron subnet, không phải từ IPAM của K8s.

4. "Cùng network plane với VM" nghĩa là gì
Hãy nhìn vào ví dụ cụ thể:
Không có Kuryr:
VM-A          IP: 192.168.1.10  (Neutron subnet)
VM-B (K8s)    IP: 192.168.1.11  (Neutron subnet)
  └─ Pod-1    IP: 10.244.0.5    (K8s internal)
  └─ Pod-2    IP: 10.244.0.6    (K8s internal)

VM-A muốn gọi Pod-1 → không thể trực tiếp
Phải đi qua NodePort hoặc LoadBalancer của K8s
VM-A chỉ biết địa chỉ 192.168.1.11:NodePort

Có Kuryr:
VM-A          IP: 192.168.1.10  (Neutron subnet)
VM-B (K8s)    IP: 192.168.1.11  (Neutron subnet)
  └─ Pod-1    IP: 192.168.1.50  (cùng Neutron subnet!)
  └─ Pod-2    IP: 192.168.1.51  (cùng Neutron subnet!)

VM-A muốn gọi Pod-1 → gọi thẳng 192.168.1.50
Packet đi qua OVS của Neutron bình thường, giống như gọi giữa 2 VM
Không cần NodePort, không cần NAT, không cần overlay của K8s


5. Tại sao không cần double encapsulation
Bình thường (Flannel trên OpenStack VM):
Pod gửi packet
  → Flannel đóng gói VXLAN (lớp 1)
  → VM gửi ra eth0
  → Neutron đóng gói VXLAN lần nữa (lớp 2)
  → Ra physical network
2 lớp tunnel = overhead lớn, latency cao.
Với Kuryr:
Pod gửi packet
  → OVS xử lý (không tunnel)
  → Neutron đóng gói VXLAN 1 lần (nếu cần)
  → Ra physical network
Chỉ 1 lớp (hoặc 0 nếu flat network) = hiệu quả hơn nhiều.

Tóm lại
CNI thườngKuryrAi cấp IP cho podK8s IPAMNeutronPod IPDải riêng của K8sNeutron subnet thậtVM gọi PodQua NodePort/LBGọi thẳng IPEncapsulationDouble (K8s + Neutron)Single (Neutron only)Độ phức tạp vận hànhĐơn giảnPhức tạp hơn

Bạn có muốn mình đi sâu vào phần nào không — ví dụ cách Kuryr map K8s Service sang Neutron LBaaS, hay cách setup thực tế?





















### 5.3 SR-IOV CNI plugin
<img width="687" height="487" alt="image" src="https://github.com/user-attachments/assets/b0fdbb6c-d79d-4376-abdf-11c81f73b016" />

- SR-IOV CNI plugin giúp passthrough thẳng VF vào eth0 của pod, không đi qua veth pair, không qua OVS bridge, không qua kernel network stack của host. Pod thấy VF như một NIC vật lý thật sự.
- Latency cực thấp và throughput cao vì packet không cần traverse qua software switching layer
- Cần hardware hỗ trợ SR-IOV


### SR-IOV (Single Root I/O Virtualization)
- SR-IOV là một tính năng phần cứng được implement trên NIC/GPU... cho phép một thiết bị vật lý (Physical Function) xuất hiện như nhiều thiết bị ảo riêng biệt (Virtual Functions) trực tiếp với các VM/container — bypassing hypervisor hoàn toàn.

- Mô hình truyền thống khi VM muốn gửi packet: VM → Hypervisor (software bridge/vswitch) → NIC vật lý . Hypervisor phải xử lý mỗi packet → overhead lớn, latency cao.

- SR-IOV giúp chia NIC vật lý thành 1 PF (Physical Function- có quyền cấu hình NIC, tạo/xóa VF) và nhiều VF (Virtual Function - chỉ xử lý data) gắn thẳng vào VM
  - PF là "toàn bộ" NIC nhìn từ phía host/hypervisor, có full quyền: cấu hình NIC, tạo VF, set MAC, VLAN, QoS
  - VF chỉ gửi/nhận packet. Mỗi VF có MAC, VLAN, queue riêng trên phần cứng. VM truy cập VF trực tiếp qua DMA mà không cần qua kernel host.

- OpenStack dùng SR-IOV để cấp VF trực tiếp cho VM (qua Neutron SR-IOV agent). Kubernetes dùng SR-IOV qua SR-IOV CNI plugin + SR-IOV Device Plugin để cấp VF cho pod

#### Use case với Kubernetes
- SR-IOV Device Plugin Chạy như DaemonSet trên mỗi node, khi phát hiện các VF available trên node nó sẽ đăng ký với kubelet như một extended resource. Pod request VF như request CPU/memory:
- SR-IOV CNI Plugin thay thế CNI mặc định (flannel/calico) cho interface đó. Khi pod được schedule → CNI plugin lấy VF đã allocated → gán vào network namespace của pod, pod thấy VF như một NIC thông thường
- Thường dùng kèm Multus vì SR-IOV CNI chỉ quản lý 1 interface, còn pod vẫn cần interface thứ nhất (eth0) cho cluster traffic → dùng Multus để attach nhiều interface cho pod

#### Use case với trong OpenStack VMs

Kiến trúc tổng quan
```
Physical Server
└── Physical NIC (PF)
    ├── VF0 ──→ OpenStack VM 1 (K8s Node)
    │           └── Pod A dùng VF này
    ├── VF1 ──→ OpenStack VM 2 (K8s Node)
    │           └── Pod B dùng VF này
    └── VF2 ──→ OpenStack VM 3 (K8s Node)
```

2 approach chính
- Approach 1: SR-IOV pass-through từ OpenStack xuống Pod: OpenStack pass-through VF thẳng vào VM → bên trong VM, Kubernetes dùng VF đó cho Pod. Ưu điểm là latency thấp nhất, gần native. Nhược điểm là VM bị bind cứng vào physical host (no live migration)

- Approach 2: DPDK/vHost trong VM: OpenStack VM dùng virtio + DPDK bên trong, Kubernetes dùng userspace CNI. Ít phổ biến và phức tạp hơn.

- Cấu hình phía OpenStack để implement SR-IOV cho pod
  - Neutron phải enable SR-IOV
  - Nova compute phải enable PCI passthrough
  - Tạo port SR-IOV trong OpenStack
  - Attach vào VM (Nova Compute + Neutron SR-IOV Agent thực thi việc gán VF cho VM) -> Bên trong VM (K8s node) sẽ thấy VF như một NIC thật
  - Từ đây setup SR-IOV CNI + Device Plugin như K8s thông thường — nhưng không cần tạo VF nữa vì VF đã được OpenStack pass vào sẵn.

Flow 
```
Bước 1: OpenStack lấy VF từ PF → gán vào VM
         (VM thấy VF như một NIC thật, ví dụ eth1)

Bước 2: Bên trong VM, K8s Device Plugin phát hiện eth1
         → đăng ký như resource

Bước 3: Pod request resource đó
         → CNI plugin gán eth1 vào network namespace của pod
```

- So sánh path của packet
  - Khi không có SR-IOV (virtio thông thường): `Pod → tap device → OVS bridge (VM) → virtio → QEMU → OVS bridge (host) → Physical NIC` (7-8 lần copy/context switch)
  - Có SR-IOV: `Pod → VF → Wire` (Gần như native, 1-2 lần)


---

OVS-CNI chỉ làm một việc: gắn pod vào một OVS bridge có sẵn trên host. Nó không:
- Gọi Neutron API
- Tạo Neutron port
- Xin IP từ Neutron subnet

Nó chỉ là dây nối pod vào OVS bridge, còn bridge đó là gì, của ai, thì OVS-CNI không quan tâm.

Nếu muốn pod nhận IP từ Neutron. OVS-CNI một mình không làm được điều đó, cần phải tự handle phần Neutron, có 2 hướng:
- Hướng 1 — Pre-provisioned port (thủ công / script)
Tự tạo Neutron port trước, rồi dùng OVS-CNI gắn pod vào port đó.
```
1. Tạo Neutron port thủ công:
   openstack port create --network my-net --fixed-ip ip-address=10.0.0.20 pod-port-1

2. Add port vào trunk của node VM:
   openstack trunk set --subport port=pod-port-1,segmentation-type=vlan,segmentation-id=100 my-trunk

3. OVS-CNI config trong pod annotation:
   gắn pod vào OVS bridge với VLAN tag 100

4. Pod lên với IP 10.0.0.20
Nhược điểm: không tự động, không scale được.
```

-Hướng 2 — Viết custom CNI / controller gọi Neutron API: Tự viết một controller (hoặc dùng Network Operator) để:
  - Watch pod creation
  - Gọi Neutron API tạo port
  - Truyền thông tin xuống OVS-CNI

---

Trunk Port là gì
- Bình thường, một VM có normal port — port này chỉ carry 1 network, traffic đi vào/ra không có VLAN tag.
- Trunk port là port đặc biệt cho phép carry nhiều network cùng lúc, mỗi network được phân biệt bằng VLAN tag.

Tại sao node VM cần trunk port
- Node VM của K8s cần chạy nhiều pod, mỗi pod có thể thuộc network Neutron khác nhau.
- Nếu dùng normal port, node VM chỉ biết 1 network → không thể phân biệt traffic của pod này với pod kia.
```
Normal port:
  Node VM  ──[port]──►  Network A only
                         (chỉ 1 network, không phân biệt được pod)

Trunk port:
  Node VM  ──[trunk]──►  Network A  (VLAN 100)  ← pod 1
                      ──►  Network B  (VLAN 200)  ← pod 2
                      ──►  Network C  (VLAN 300)  ← pod 3
```

Flow cụ thể
Không có trunk — traffic bị chặn
```
Pod (IP: 10.0.0.20, Network A)
  └─ veth
      └─ OVS bridge trên node
          └─ normal port của node VM
              └─ OVN thấy packet từ MAC lạ (không phải VM)
                  └─ DROP  ❌
OVN drop vì MAC của pod không được đăng ký trên port đó — port spoofing protection.
```

Có trunk — traffic đi được
```
Pod (IP: 10.0.0.20, Network A)
  └─ veth
      └─ OVS bridge trên node
          └─ VLAN tag 100 được gắn vào packet
              └─ trunk port của node VM
                  └─ OVN nhận packet, thấy VLAN 100
                      └─ map sang sub-port của Network A  ✅
                          └─ forward bình thường
```

Sub-port là gì: Trunk có 1 parent port (port chính của node VM) và nhiều sub-port (mỗi sub-port = 1 Neutron port của pod).
```
Trunk
├── parent-port  →  node VM eth0  (IP: 192.168.1.10, không tag)
├── sub-port A   →  pod-1         (IP: 10.0.0.20, VLAN 100)
└── sub-port B   →  pod-2         (IP: 10.0.1.30, VLAN 200)
```
Khi add pod port vào trunk:
```
openstack trunk set \
  --subport port=<pod-neutron-port-id>,\
            segmentation-type=vlan,\
            segmentation-id=100 \
  <trunk-name>
```
Lệnh này nói với OVN: "traffic VLAN 100 đi qua trunk này là của pod-port-id đó".

Tk tpbank: Ls2v3honey@
Tk Techcombank: Vis2022 mã 180520
Tk MBBank: ngocmai95
Tk BIdv: Ls2v3honey@
Tk vpbank: Visvietnam1805@
