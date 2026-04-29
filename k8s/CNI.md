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

## 3. Bridge 
- Là một virtual network switch được implement trong Linux kernel, có chức năng giống như một switch vật lý Layer 2 nhưng hoàn toàn bằng software. Thay vì port vật lý, bridge có các network interface gắn vào làm port. Đối với host Linux bridge xuất hiện là một interface trong host network namespace, giống như bất kỳ interface nào khác (eth0, lo...). Có thể show bằng lệnh `ip link show` → Sẽ thấy cni0/docker0 nằm chung với eth0, lo, flannel.1...
- Docker và K8s sử dụng Linux bridge để implement network vì nó có sẵn trong kernel, không cần cài thêm gì. Trong Docker bridge thường có tên là `docker0` do Docker daemon tạo ra. Còn trong Kubernetes thì bridge do CNI tạo và có tên là `cni0`. Openstack sử dụng OVS làm bridge




### 3.1. Linux bridge
- Các CNI plugin phổ biến tạo Linux Bridge tên cni0 trên mỗi node. Bridge này hoạt động như một virtual switch Layer 2, kết nối các veth pair của tất cả pod trên cùng node lại với nhau.
- Khi 1 pod trong cụm k8s được tạo ra thì CNI plugin tạo một veth pair — một đầu gắn vào network namespace của pod, đầu còn lại (là các veth trên host) gắn vào bridge cni0.
- Linux bridge là một interface trong host network namespace, giống như bất kỳ interface nào khác (eth0, lo...). Khi liệt kê interface bằng lệnh `ip link show` sẽ thấy cni0/docker0 nằm chung với eth0, lo, flannel.1...
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
- Ngoài cni0 trong K8s có thể có cni1, cni2 nếu dùng Multus CNI cho phép pod có nhiều network interface. Khi đó bạn có thể define nhiều NetworkAttachmentDefinition, mỗi cái trỏ tới một bridge plugin config với tên bridge khác nhau
- Lưu ý không phải CNI plugin nào cũng dùng bridge. Ví dụ Calico gắn veth trực tiếp vào routing table của host (cần kernel xử lý L3 routing cho từng pod). Cilium thì dùng eBPF, có thể bypass bridge hoàn toàn. Với OVS-based CNI traffic đi qua OVS bridge thay vì Linux bridge

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

- OVS-CNI là một CNI plugin cho phép Kubernetes pod gắn network interface trực tiếp vào Open vSwitch (OVS) bridge, thay vì dùng Linux bridge
- Có thể tận dụng OVS-CNI plugin để kết nối pod network với Neutron OVS backend
- Lưu ý OVS-CNI không tự tạo OVS bridge. Nó expect bridge đã tồn tại sẵn trên **K8s node (tức VM)**. Không thể tận dụng OVS bridge của Openstack hypervisor bên dưới do K8s node chỉ thấy eth0 - một virtual NIC bình thường. Nó không có visibility vào lớp OpenStack hypervisor.
- Khi bạn tạo một VM trên OpenStack:
  - Nova yêu cầu Neutron: "Tạo cho tôi một network interface"
  - Neutron tạo một "Port" (logical) trong database
  - Neutron agent trên hypervisor node nhận event
  - Agent gọi ovs-vsctl để tạo port thật trên OVS bridge
  - VM được cắm vào OVS bridge qua tap interface
- Khi pod được tạo, Hyperplane plugin (custom CNI của Samsung) làm:
  - Kubelet gọi Hyperplane CNI plugin
  - CNI plugin gọi Neutron API: "Tạo cho tôi một Port trong subnet X"
  - Neutron tạo Port, trả về: IP = 192.168.1.50, MAC = aa:bb:cc:...
  - Neutron agent trên node tạo OVS port thật (giống như tạo cho VM)
  - Plugin gắn OVS port đó vào network namespace của pod -> Pod bây giờ có IP 192.168.1.50 — chính là IP từ Neutron subnet, không phải từ IPAM của K8s. Pod bây giờ có cùng network với VM, VM-A muốn gọi Pod-1 → gọi thẳng 192.168.1.50. Packet đi qua OVS của Neutron bình thường, giống như gọi giữa 2 VM, không cần NodePort, không cần NAT, không cần overlay của K8s

====================================

Flow hoạt động:
- Khi pod được schedule lên VM node, kubelet tạo một network namespace mới (thực chất là một ip netns riêng biệt). Lúc này namespace đó chỉ có interface lo — hoàn toàn cô lập, không thấy gì bên ngoài.
- CNI plugin chạy như một process trên VM node với quyền root, có toàn quyền truy cập vào OVS br-int trong VM, ovn-controller trong VM, tất cả network namespace trong VM và Neutron API (gọi qua network)
- Khi kubelet gọi CNI plugin (Hyperplane)
Plugin gọi Neutron API → nhận lại thông tin port (IP, MAC, port UUID)
Neutron agent (OVS agent) trên host tạo một OVS internal port hoặc veth pair, gắn vào br-int
Plugin dùng lệnh kiểu ip link set <veth-end> netns <pod-namespace> để đẩy một đầu của interface vào namespace của pod
Bên trong namespace đó, plugin cấu hình IP, MAC, default route — cũng bằng ip netns exec từ host

Pod không chủ động trong quá trình này. Từ góc nhìn của pod, nó "tỉnh dậy" và thấy mình đã có sẵn một interface với IP, MAC, route — giống như VM boot lên đã có NIC sẵn vậy. Pod không biết Neutron là gì, không gọi API nào, không kéo port nào cả.
Nói cách khác: CNI plugin đóng vai trò giống như nova-compute gắn NIC vào VM — chỉ là ở đây "VM" là một network namespace của pod, và "hypervisor" là chính Linux kernel trên host.You said: trường external_ids là metadata của cái gìtrường external_ids là metadata của cái gìApr 28Claude responded: Cần thêm ngữ cảnh — external_ids xuất hiện ở nhiều chỗ khác nhau và ý nghĩa khác nhau:Cần thêm ngữ cảnh — external_ids xuất hiện ở nhiều chỗ khác nhau và ý nghĩa khác nhau:
Trong OVS (Open vSwitch): external_ids là một trường metadata dạng key-value gắn trên các object của OVS — port, bridge, interface. Ví dụ khi chạy ovs-vsctl show hay ovs-vsctl get Interface <name> external_ids, sẽ thấy những cặp như iface-id, attached-mac… Neutron OVS agent dùng trường này để map một OVS port với một Neutron port UUID.
Trong Neutron port: Neutron port cũng có trường external_ids hoặc tương đương (thường gọi là binding:profile, binding:vif_details…) để lưu metadata liên kết với hạ tầng bên dưới.
Bạn đang hỏi external_ids ở đâu cụ thể — trên OVS port, trên Neutron port, hay trong một output/config nào đó? Gửi đoạn output hoặc context thì mình giải thích chính xác hơn.You said: trên ovs porttrên ovs portApr 28Claude responded: OK, vậy external_ids trên OVS port là metadata mà Neutron OVS agent ghi vào để liên kết một OVS port với một Neutron port.OK, vậy external_ids trên OVS port là metadata mà Neutron OVS agent ghi vào để liên kết một OVS port với một Neutron port.
Bản thân OVS không quan tâm đến trường này — OVS chỉ lưu nó như một cái túi key-value. Neutron agent mới là thằng đọc/ghi và dựa vào đó để làm việc.
Ví dụ điển hình khi chạy ovs-vsctl get Interface tap-xxx external_ids:
{iface-id="neutron-port-uuid-abc123",
 iface-status="active",
 attached-mac="fa:16:3e:xx:xx:xx"}
Ý nghĩa từng key phổ biến:

iface-id — quan trọng nhất. Đây là Neutron port UUID. Neutron OVS agent dùng giá trị này để biết "OVS port này thuộc về Neutron port nào" → từ đó áp đúng security group, flow rules, VLAN/VXLAN tag…
iface-status — trạng thái active/inactive, agent cập nhật ngược lại cho Neutron biết port đã sống hay chưa
attached-mac — MAC address mà Neutron đã cấp cho port đó

Trong ngữ cảnh Hyperplane CNI, luồng sẽ là: sau khi Neutron tạo port và trả về UUID, CNI plugin (hoặc Neutron agent) tạo OVS port trên br-int và set external_ids:iface-id=<neutron-port-uuid>. Nhờ đó OVS agent nhận ra port này, apply đúng flow rules, và báo lại Neutron rằng port đã active.
Nói đơn giản: external_ids là cách Neutron "dán nhãn" lên OVS port để nhận ra đứa con của mình.You said: Nếu tôi dùng ovn thì ai phụ trách việc ghi đấyNếu tôi dùng ovn thì ai phụ trách việc ghi đấyApr 28Claude responded: Với OVN thì kiến trúc khác so với OVS agent truyền thống.Với OVN thì kiến trúc khác so với OVS agent truyền thống.
ovn-controller trên mỗi node là thằng phụ trách việc này.
Luồng cụ thể:

Neutron tạo port → ghi thông tin vào OVN Northbound DB (logical switch port)
ovn-northd dịch từ Northbound DB → Southbound DB (binding, flow rules…)
ovn-controller trên node đọc Southbound DB, thấy có logical port cần bind trên node này
ovn-controller tìm OVS port nào có external_ids:iface-id khớp với logical port đó → bind và cài OpenFlow rules vào br-int

Nhưng ai ghi external_ids:iface-id lên OVS port? Đây là điểm hay — không phải ovn-controller ghi. Thằng ghi là bất kỳ ai tạo OVS port đó:

Với VM: nova-compute (thông qua libvirt) tạo tap interface và set external_ids:iface-id=<neutron-port-uuid>
Với Hyperplane CNI: CNI plugin tạo OVS port và set external_ids:iface-id

ovn-controller chỉ đọc external_ids:iface-id, match với logical port trong Southbound DB, rồi claim binding và install flow. Nó không tự ghi trường đó.
Tóm lại phân công rõ ràng: CNI plugin (hoặc nova) ghi nhãn, ovn-controller đọc nhãn và hành động.You said: external_ids trên OVS port là metadata mà Neutron OVS agent ghi vào để liên kết một OVS port với một Neutron port.external_ids trên OVS port là metadata mà Neutron OVS agent ghi vào để liên kết một OVS port với một Neutron port. -> ý b ghi vào là ghi vào đâuApr 28Claude responded: Đúng, câu trước mình diễn đạt dễ gây hiểu nhầm.Đúng, câu trước mình diễn đạt dễ gây hiểu nhầm.
"Ghi vào" ở đây nghĩa là ghi vào OVSDB — cơ sở dữ liệu của Open vSwitch chạy trên chính node đó.
Mỗi node chạy OVS đều có một ovsdb-server process, lưu toàn bộ cấu hình của OVS (bridge, port, interface, external_ids…). Khi ai đó chạy lệnh kiểu ovs-vsctl set Interface tap-xxx external_ids:iface-id=<uuid>, thực chất là ghi một record vào OVSDB trên node đó.
Nên câu chính xác hơn phải là: Neutron OVS agent (hoặc CNI plugin, hoặc nova) ghi trường external_ids vào OVSDB trên node, để liên kết một OVS port với một Neutron port.You said: tại sao cần liên kết một OVS port với một Neutron port.tại sao cần liên kết một OVS port với một Neutron port.12:01 AMClaude responded: Vì OVS và Neutron là hai thằng riêng biệt, mỗi thằng quản lý một "thế giới" khác nhau.Vì OVS và Neutron là hai thằng riêng biệt, mỗi thằng quản lý một "thế giới" khác nhau.
OVS chỉ biết: "tôi có một port tên tap-xxx, nó gắn vào br-int, packet vào ra qua đây." Hết. OVS không biết port đó thuộc tenant nào, subnet nào, security group nào.
Neutron chỉ biết: "tôi có một logical port UUID abc-123, IP là 192.168.1.50, thuộc subnet X, tenant Y, áp security group Z." Nhưng Neutron không trực tiếp đụng vào dataplane — nó không tự cài flow rules.
Nếu không liên kết, ovn-controller (hoặc OVS agent) nhìn thấy một OVS port nhưng không biết nó là ai trong "thế giới" của Neutron → không biết áp security group nào, không biết gắn vào logical switch nào, không biết VXLAN tunnel nào cần forward traffic đến. Port đó thành một cái lỗ câm trên bridge.
Khi có external_ids:iface-id=abc-123, ovn-controller đọc được: "à, OVS port này chính là Neutron port abc-123" → tra Southbound DB → biết port thuộc logical switch nào → cài đúng OpenFlow rules → traffic đi đúng chỗ, security group được enforce, mọi thứ hoạt động.
Nói ngắn gọn: external_ids là cầu nối giữa dataplane (OVS) và control plane (Neutron). Thiếu nó thì hai thằng không nhận ra nhau.You said: hành vi liên kết một OVS port với một Neutron port là chung hay chỉ mỗi hyperplane cni cần làmhành vi liên kết một OVS port với một Neutron port là chung hay chỉ mỗi hyperplane cni cần làm12:03 AMClaude responded: Đây là hành vi chung, ai tạo OVS port cũng phải làm.Đây là hành vi chung, ai tạo OVS port cũng phải làm.

VM qua Nova: nova-compute gọi libvirt tạo tap interface → set external_ids:iface-id → ovn-controller đọc và bind
Kuryr CNI (pod): Kuryr tạo OVS port cho pod → set external_ids:iface-id → ovn-controller đọc và bind
Hyperplane CNI (pod): tương tự, tạo OVS port → set external_ids:iface-id
LBaaS (Octavia): amphora VM cũng có Neutron port → cùng cơ chế

Bất kỳ thứ gì muốn tham gia vào mạng của Neutron/OVN đều phải làm bước này. Đây không phải đặc thù của Hyperplane — đây là quy ước chung của kiến trúc Neutron + OVS/OVN. Không set iface-id thì ovn-controller không nhận ra port, coi như port đó không tồn tại trong thế giới Neutron.You said: Đấy có phải là cơ chế port binding koĐấy có phải là cơ chế port binding ko12:39 AMClaude responded: Đúng, đó chính là một phần của cơ chế port binding.Đúng, đó chính là một phần của cơ chế port binding.
Nhưng port binding không chỉ là set external_ids. Nó là cả một quy trình gồm nhiều bước:

Neutron tạo logical port → trạng thái DOWN, chưa bind ở đâu
Ai đó (nova, CNI…) gọi Neutron API báo: "tôi muốn bind port này trên host X" → Neutron ghi binding:host_id=host-X vào port
Neutron ML2 plugin chọn mechanism driver phù hợp (OVN, OVS agent, SRIOV…) để thực hiện binding ở tầng backend
Trên node, OVS port được tạo với external_ids:iface-id → đây là bước mà ovn-controller (hoặc OVS agent) nhận ra port và hoàn tất binding
ovn-controller claim port trong Southbound DB → Neutron cập nhật trạng thái port thành ACTIVE

Nên external_ids:iface-id là bước cuối cùng trên dataplane để hoàn tất port binding — là cái "bắt tay" giữa control plane và dataplane. Còn port binding tổng thể bao gồm cả phần API, scheduling, và ML2 driver phía trên nữa.You said: OVS chỉ biết: “tôi có một port tên tap-xxx, nó gắn vào br-int, packet vào ra qua đây -> bạn giải thích cho tôi ý này.OVS chỉ biết: “tôi có một port tên tap-xxx, nó gắn vào br-int, packet vào ra qua đây -> bạn giải thích cho tôi ý này. Ovs bạn nói đến ở đây là agent hay ovs bridge1:03 AMClaude responded: Ở đây mình nói đến OVS bridge (dataplane) — tức bản thân br-int, br-ex… chạy trên node.Ở đây mình nói đến OVS bridge (dataplane) — tức bản thân br-int, br-ex… chạy trên node.
OVS bridge thực chất là một software switch. Nó hoạt động giống switch vật lý — chỉ biết:

Tôi có những port nào (tap-xxx, patch-br-ex, veth-yyy…)
Packet vào từ port nào, match flow rule nào, forward ra port nào

Nó không biết và không quan tâm: port đó thuộc tenant gì, subnet gì, security group gì. Những khái niệm đó không tồn tại ở tầng dataplane.
Còn OVS agent (hoặc ovn-controller) là tầng trên — nó đọc thông tin từ Neutron/OVN, rồi cài flow rules xuống OVS bridge để bridge biết cách xử lý packet. Ví dụ: "packet từ port tap-xxx mà MAC không đúng thì drop" — đó là security group, nhưng bridge chỉ thấy nó như một flow rule, không biết khái niệm security group là gì.
Tóm lại: OVS bridge là thằng chuyển packet nhưng ngu — cần OVS agent hoặc ovn-controller lập trình cho nó biết chuyển thế nào.You said: Đầu cắm vào bridge gọi là tap interface à bĐầu cắm vào bridge gọi là tap interface à b1:03 AMClaude responded: Không hẳn.Không hẳn. tap chỉ là một loại interface — không phải mọi thứ cắm vào bridge đều là tap.
Những loại interface phổ biến gắn vào br-int:

tap interface — thường dùng cho VM (libvirt tạo tap, gắn vào bridge, đầu kia nối vào QEMU process)
veth pair — thường dùng cho container/pod (một đầu gắn vào bridge, đầu kia đẩy vào network namespace của pod)
patch port — nối giữa hai OVS bridge với nhau (ví dụ br-int ↔ br-ex)
internal port — OVS tự tạo, thường dùng làm gateway interface
tunnel port (VXLAN, Geneve…) — đường hầm nối giữa các node

Cái chung là tất cả đều gọi là OVS port khi đã gắn vào bridge. "Tap interface" chỉ là tên gọi một kiểu cụ thể, xuất phát từ thời VM dùng tap device trong Linux kernel.
Trong ngữ cảnh Hyperplane CNI, khả năng cao nó dùng veth pair — không phải tap.You said: Khi có external_ids:iface-id=abc-123, ovn-controller đọc được: “à, OVS port này chính là Neutron port abc-123” → tra Southbound DB → biết port thuộc logical sw…Khi có external_ids:iface-id=abc-123, ovn-controller đọc được: “à, OVS port này chính là Neutron port abc-123” → tra Southbound DB → biết port thuộc logical switch nào -> logical switch bạn nhắc đến ở đây là gì1:09 AMClaude responded: Logical switch là khái niệm của OVN — nó là một switch ảo "tưởng tượng" tồn tại trong database, không phải trên dataplane.Logical switch là khái niệm của OVN — nó là một switch ảo "tưởng tượng" tồn tại trong database, không phải trên dataplane.
Khi bạn tạo một Neutron network, Neutron (thông qua ML2/OVN driver) tạo tương ứng một logical switch trong OVN Northbound DB. Mỗi Neutron network = một logical switch.
Ví dụ: bạn tạo network "app-network" subnet 192.168.1.0/24 → OVN Northbound DB có một logical switch tên tương ứng. Mọi Neutron port thuộc network đó → trở thành logical switch port gắn vào logical switch đó.
Tại sao cần khái niệm này? Vì trên thực tế, các VM/pod nằm rải rác trên nhiều node khác nhau, mỗi node có br-int riêng. Logical switch là cách OVN nói: "dù port A ở node 1 và port B ở node 3, nhưng cả hai đều nằm trên cùng một switch ảo → chúng có thể giao tiếp L2 với nhau."
ovn-controller trên mỗi node đọc logical switch từ Southbound DB, rồi dịch thành OpenFlow rules cài vào br-int thật trên node đó — bao gồm cả việc đóng gói Geneve tunnel để chuyển packet giữa các node nếu cần.
Nói ngắn gọn: logical switch là bản thiết kế, br-int + flow rules là hiện thực hóa bản thiết kế đó trên từng node.You said: Tại sao các port lại nằm ở các node khác nhau được.Tại sao các port lại nằm ở các node khác nhau được. Tôi tưởng port là khái niệm trừu tượng thuộc về control plane chứ1:12 AMClaude responded: Đúng, Neutron port / logical switch port là khái niệm trừu tượng ở control plane.Đúng, Neutron port / logical switch port là khái niệm trừu tượng ở control plane. Nhưng mỗi logical port phải bind xuống một node cụ thể — vì nó đại diện cho một VM hoặc pod thật, mà VM/pod thì phải chạy trên một node nào đó.
Ví dụ cụ thể:

Pod-A chạy trên node-1 → Neutron port aaa-111 bind trên node-1 → OVS port trên br-int của node-1
Pod-B chạy trên node-3 → Neutron port bbb-222 bind trên node-3 → OVS port trên br-int của node-3

Cả hai port đều thuộc cùng một logical switch (cùng Neutron network). Ở tầng control plane thì chúng "ngồi cạnh nhau" trên cùng một switch ảo. Nhưng ở tầng dataplane, chúng hiện thực hóa trên hai node khác nhau.
Nên khi mình nói "port nằm ở các node khác nhau", ý là phần dataplane của port (OVS port thật, veth, tap…) nằm ở các node khác nhau. Còn phần logical thì đúng như bạn nói — nó là trừu tượng, sống trong database.
Chính vì vậy mới cần Geneve/VXLAN tunnel giữa các node — để hai OVS port trên hai br-int khác node có thể giao tiếp được với nhau, tạo ảo giác rằng chúng cùng nằm trên một switch.You said: Lệnh ovs-vsctl là để làm việc với cái gì.Lệnh ovs-vsctl là để làm việc với cái gì. Nếu dùng ovn thì thay bằng lệnh gì1:23 AMClaude responded: ovs-vsctl là để làm việc với OVSDB — tức cấu hình của OVS trên node đó (tạo bridge, thêm port, set external_ids…).ovs-vsctl là để làm việc với OVSDB — tức cấu hình của OVS trên node đó (tạo bridge, thêm port, set external_ids…).
Khi dùng OVN, ovs-vsctl vẫn dùng vì OVN không thay thế OVS — OVN chạy phía trên OVS. Trên mỗi node vẫn có br-int, vẫn có OVSDB, vẫn cần ovs-vsctl để quản lý tầng dataplane.
OVN thêm vào một bộ lệnh riêng để quản lý tầng logical:

ovn-nbctl — làm việc với Northbound DB (logical switch, logical router, ACL, NAT…). Đây là tầng gần với Neutron, mô tả "mạng trông như thế nào."
ovn-sbctl — làm việc với Southbound DB (port binding, datapath, flow logic…). Đây là tầng trung gian, mô tả "mạng được hiện thực hóa thế nào trên từng node."

Tóm lại:

Muốn xem/sửa OVS bridge, port, interface trên node → ovs-vsctl (vẫn dùng bình thường)
Muốn xem/tạo logical switch, router, ACL → ovn-nbctl
Muốn xem port binding, chassis, flow → ovn-sbctl
Muốn xem OpenFlow rules thật trên bridge → ovs-ofctl (cũng vẫn dùng)

OVN không thay ovs-vsctl — nó bổ sung thêm ovn-nbctl và ovn-sbctl để quản lý tầng logic phía trên.You said: Vậy nếu tôi dùng veth thì lệnh này sẽ đổi thành ovs-vsctl get Interface veth-xxx external_ids phải koVậy nếu tôi dùng veth thì lệnh này sẽ đổi thành ovs-vsctl get Interface veth-xxx external_ids phải ko1:27 AMClaude responded: Đúng, đổi tên interface thôi.Đúng, đổi tên interface thôi. Cú pháp y hệt:
ovs-vsctl get Interface veth-xxx external_ids
Tên interface ở đây là tên của đầu veth gắn vào br-int — đầu còn lại nằm trong network namespace của pod, không liên quan đến OVS.
Tên cụ thể là gì thì tùy CNI plugin đặt. Có thể là veth-xxx, abcdef123, hay bất kỳ cái gì — miễn là trùng với tên port đã add vào bridge. Bạn chạy ovs-vsctl list-ports br-int sẽ thấy tên thật.You said: Nếu veth pair của pod cắm vào ovs bridge thì nó phải đi qua mấy lớp mạng à, có khả thi ko bạnNếu veth pair của pod cắm vào ovs bridge thì nó phải đi qua mấy lớp mạng à, có khả thi ko bạn12:37 PMClaude responded: Hoàn toàn khả thi và rất phổ biến.Hoàn toàn khả thi và rất phổ biến. Nhưng mình hiểu câu hỏi của bạn — bạn lo ngại veth pair cắm vào OVS bridge thì packet phải đi qua nhiều lớp xử lý.
Thực tế thì đường đi rất ngắn:
Pod → veth (trong namespace) → veth (trên host) → OVS br-int → flow rules xử lý → ra ngoài
Chỉ vậy thôi. veth pair chỉ là một cái ống nối giữa hai network namespace — packet vào đầu này thì xuất hiện ngay đầu kia, không qua xử lý gì ở giữa. Nó không thêm "lớp mạng" nào cả, chỉ là cơ chế chuyển packet giữa namespace.
So sánh với VM dùng tap: VM → tap interface → OVS br-int. Cũng một bước tương tự. veth không nặng hơn tap đáng kể.
Cái tốn performance thật sự không phải veth mà là số lượng OpenFlow rules trên br-int — nhiều security group, nhiều ACL thì flow table lớn, lookup chậm hơn. Nhưng đó là vấn đề chung dù dùng tap hay veth.
Nếu thật sự quan tâm performance, giải pháp là SR-IOV — bypass luôn cả OVS, gắn VF thẳng vào pod. Nhưng mất đi software-defined networking (security group, flow rules…). Đó là trade-off mà bạn đã tìm hiểu trước đó rồi.You said: Theo t nó từ veth còn phải đi qua tap interface rồi mới vào ovs brint nữa chứTheo t nó từ veth còn phải đi qua tap interface rồi mới vào ovs brint nữa chứ12:38 PMClaude responded: Không, veth không cần đi qua tap.Không, veth không cần đi qua tap. Đây là hai cơ chế riêng biệt, dùng một trong hai:
Cách 1 — VM dùng tap:
VM (QEMU process) → tap interface → OVS br-int
Cách 2 — Pod dùng veth:
Pod (network namespace) → veth end trong ns → veth end trên host → OVS br-int
Không có chuyện veth → tap → OVS. Đầu veth trên host gắn thẳng vào br-int luôn bằng lệnh kiểu:
ovs-vsctl add-port br-int veth-host-end
tap interface chỉ cần khi có QEMU process — vì QEMU đọc/ghi packet thông qua file descriptor của tap device. Pod không chạy QEMU, không cần tap.
Có thể bạn đang nhầm với một số CNI setup dùng Linux bridge làm trung gian: veth → Linux bridge → tap → OVS. Đó là kiến trúc cũ của Neutron (hybrid firewall driver) khi cần iptables cho security group. Nhưng với OVN thì không cần Linux bridge trung gian — security group được xử lý bằng OpenFlow rules ngay trên br-int, nên veth cắm thẳng vào OVS được.You said: veth end trên host thì host ở đây là VM hay compute nodeveth end trên host thì host ở đây là VM hay compute node12:39 PMClaude responded: Tùy kiến trúc của bạn.Tùy kiến trúc của bạn.
Nếu K8s chạy bare-metal — host là compute node vật lý. veth end nằm trên root network namespace của máy vật lý đó.
Nếu K8s chạy nested trong VM trên OpenStack — như lab của bạn đang làm — thì host là VM đóng vai trò K8s node. veth end nằm trên root network namespace của VM đó.
Trong trường hợp nested của bạn, đường đi đầy đủ sẽ dài hơn:
Pod → veth → OVS br-int (trong VM) → VM's NIC → OVS br-int (trên compute node vật lý) → ra ngoài
Tức là có hai tầng OVS — một tầng trong VM, một tầng trên compute node. Đây cũng là lý do nested virtualization chậm hơn bare-metal đáng kể — packet phải đi qua hai lần OVS processing.You said: vậy khi dùng ovs cni là cắm veth vào ovs nàovậy khi dùng ovs cni là cắm veth vào ovs nào1:16 PMClaude responded: Cắm vào OVS br-int bên trong VM (VM đóng vai K8s node).Cắm vào OVS br-int bên trong VM (VM đóng vai K8s node).
Vì CNI plugin chạy trên K8s node — mà K8s node của bạn là VM — nên nó chỉ thấy và chỉ làm việc với OVS instance bên trong VM đó. Nó không biết và không đụng được OVS trên compute node vật lý bên ngoài.
Nhưng đây là điểm cần suy nghĩ trong lab nested của bạn: VM đó có sẵn OVS br-int không?

Nếu VM chỉ là K8s node thuần — không chạy Neutron agent hay ovn-controller bên trong — thì không có sẵn OVS. CNI plugin phải tự cài OVS, tự tạo bridge.
Nếu bạn cài cả OpenStack bên trong (nested OpenStack) hoặc cài ovn-controller trên VM đó — thì có br-int sẵn, CNI cắm vào đó được.

Trong kiến trúc Hyperplane mà bạn mô tả ban đầu — pod dùng Neutron port, muốn ovn-controller bind port — thì VM (K8s node) phải chạy ovn-controller và có br-int. Lúc đó veth cắm vào br-int này, ovn-controller đọc external_ids, mọi thứ hoạt động như đã thảo luận.You said: vậy Trong kiến trúc Hyperplane mà tôi mô tả ban đầu thì sẽ có 2 ovn-controller phải ko - 1 trong VM và 1 trong openstack node, đồng thời có 2 br-int phải kovậy Trong kiến trúc Hyperplane mà tôi mô tả ban đầu thì sẽ có 2 ovn-controller phải ko - 1 trong VM và 1 trong openstack node, đồng thời có 2 br-int phải ko1:20 PMClaude responded: Đúng, trong nested setup của bạn thì đúng là 2 ovn-controller và 2 br-int.Đúng, trong nested setup của bạn thì đúng là 2 ovn-controller và 2 br-int.
Nhưng câu hỏi quan trọng là: chúng thuộc cùng một OVN cluster hay hai cluster riêng?
Nếu cùng một OVN cluster — ovn-controller trong VM kết nối tới cùng Southbound DB với ovn-controller trên compute node. Lúc này Neutron quản lý cả VM port lẫn pod port trong cùng một hệ thống. Pod thật sự ngang hàng với VM trên cùng logical switch. Đây là kiến trúc đúng nghĩa Hyperplane.
Nếu hai OVN cluster riêng — ví dụ OpenStack ngoài có OVN riêng, K8s trong VM có OVN riêng — thì hai tầng br-int không biết nhau. Pod không thể ngang hàng với VM trên cùng Neutron network. Mất đi ý nghĩa của Hyperplane.
Nên để Hyperplane hoạt động đúng trong nested setup, ovn-controller trong VM phải join vào OVN cluster của OpenStack bên ngoài — trỏ Southbound DB remote tới cùng chỗ. Lúc đó tuy có 2 br-int vật lý trên 2 tầng, nhưng chúng được điều phối bởi cùng một control plane, và pod traffic từ br-int trong VM → qua VM NIC → br-int ngoài compute node → tới đích, tất cả do cùng một OVN cluster quản lý.You said: tôi muốn chúng cùng 1 ovn cluster để pod ngang hàng với VM thì phải làm thế nàotôi muốn chúng cùng 1 ovn cluster để pod ngang hàng với VM thì phải làm thế nào1:21 PMClaude responded: Về nguyên tắc, bạn cần cài ovn-controller trong VM và trỏ nó về Southbound DB của OpenStack bên ngoài.Về nguyên tắc, bạn cần cài ovn-controller trong VM và trỏ nó về Southbound DB của OpenStack bên ngoài.
Cụ thể:
1. Trên VM (K8s node), cài OVS + ovn-controller:
bashapt install ovn-host openvswitch-switch
2. Trỏ ovn-controller về Southbound DB bên ngoài:
bashovs-vsctl set open_vswitch . \
  external_ids:ovn-remote=tcp:<southbound-db-ip>:6642 \
  external_ids:ovn-encap-type=geneve \
  external_ids:ovn-encap-ip=<VM-ip-trên-provider-network> \
  external_ids:system-id=<unique-id-cho-VM>
Trong đó:

ovn-remote — địa chỉ Southbound DB, chạy trên controller node của OpenStack
ovn-encap-ip — IP mà các node khác dùng để gửi Geneve tunnel tới VM này. Phải là IP mà compute node bên ngoài reach được
system-id — chassis ID unique, để OVN nhận ra VM này là một "node" trong cluster

3. Khi ovn-controller khởi động, nó tự đăng ký như một chassis mới trong Southbound DB. Từ đây OVN coi VM này ngang hàng với compute node — có thể bind logical port, nhận flow rules.
4. CNI plugin tạo pod → gọi Neutron tạo port → tạo veth gắn vào br-int trong VM → set external_ids:iface-id → ovn-controller trong VM claim binding → pod online trên Neutron network.
Vấn đề cần lưu ý với nested:
Geneve tunnel lồng nhau — traffic từ pod trong VM muốn ra ngoài phải đóng Geneve (bởi ovn-controller trong VM), rồi packet đó đi qua VM NIC, tới br-int trên compute node, có thể lại đóng thêm một lớp Geneve nữa. Đây là overhead lớn nhất của nested setup. Bạn cần đảm bảo MTU đủ lớn để chứa double encapsulation, hoặc dùng provider network (flat/VLAN) cho VM để tránh lớp tunnel ngoài.





















================================
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
