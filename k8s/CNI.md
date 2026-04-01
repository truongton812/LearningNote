CNI (Container Network Interface) là một đặc tả (specification) và bộ thư viện dùng để cấu hình network interfaces cho các container trong Linux.

CNI định nghĩa một giao diện chuẩn giữa container runtime (như Kubernetes, Podman) và các network plugin, giúp chúng hoạt động với nhau mà không cần phụ thuộc vào nhau.

Các thành phần chính của CNI
1. Specification (Đặc tả): Định nghĩa cách một plugin nhận input và trả output — thông qua biến môi trường và stdin/stdout theo định dạng JSON.
2. Plugin: Là các file thực thi (executables) thực hiện việc cấu hình mạng. Có hai loại:
- Interface plugin: tạo network interface (ví dụ: bridge, macvlan, ipvlan)
- Chained plugin: bổ sung thêm tính năng (ví dụ: portmap, bandwidth, firewall)

3. Library (libcni): Thư viện Go giúp tích hợp CNI vào container runtime dễ dàng hơn.

---
### CNI specification

The CNI specification defines:
- Định dạng chuẩn (thường là file JSON/YAML) để admin định nghĩa cấu hình mạng
- Giao thức chuẩn (gọi qua command-line như plugin_linux ADD/DEL) để runtime (containerd, CRI-O, Docker) gửi request tới plugin CNI.
- Quy trình để thực thi các plugins dựa trên network configuration ở trên
- Quy trình để plugin delegate (gọi nhúng) plugin con để xử lý phần việc, ví dụ: host-local delegate cho IPAM, tạo chain linh hoạt.
- Định dạng JSON chuẩn cho plugin trả kết quả cho runtime

#### Ví dụ thực tế

Giả sử cluster dùng CNI plugin là Calico để cấp IP cho pod và container runtime là containerd

Mỗi khi bạn chạy lệnh: kubectl run nginx --image=nginx. Kubernetes sẽ: Tạo một pod. Sau đó yêu cầu container runtime (ở đây là containerd) khởi tạo container. Container runtime sẽ liên hệ với CNI plugin để “gắn” mạng cho container đó.

1. Định dạng chuẩn (thường là file JSON/YAML) để admin định nghĩa cấu hình mạng
Đây là file config CNI mà admin (bạn) định nghĩa, ví dụ 1 file như 10-calico.conflist:
```
{
  "cniVersion": "0.8.0",
  "name": "mynet",
  "plugins": [
    {
      "type": "calico",
      "etcd_endpoints": "https://10.0.0.3:2379",
      "subnet": "192.168.0.0/16"
    }
  ]
}
```

Ý nghĩa: Bạn nói với CNI:
- Mạng này tên là mynet.
- Dùng plugin calico.
- Dải IP cho pod là 192.168.0.0/16.

2. Giao thức chuẩn để runtime gửi request tới plugin CNI.
Khi runtime (containerd) muốn “gắn mạng” cho container nginx, nó sẽ:
- Đọc file config CNI ở trên.
- Gọi file thực thi plugin CNI theo chuẩn, ví dụ: `/opt/cni/bin/calico ADD` -> Dữ liệu config (như JSON bên trên) được đưa vào stdin của lệnh này

3. Quy trình để thực thi các plugins dựa trên network configuration

Các plugin thực thi theo quy chuẩn CNI đặt ra. Ví dụ:
- Tạo veth pair trong container.
- Gán IP từ dải mới cho container.
- Cập nhật iptables/routes nếu cần.

4.  Quy trình để plugin delegate (gọi nhúng) plugin con để xử lý phần việc
Giả sử config CNI của bạn là file conflist (nhiều plugin):
```
{
  "cniVersion": "0.8.0",
  "name": "mynet",
  "plugins": [
    {
      "type": "calico" # calico plugin
    },
    {
      "type": "portmap", #portmap plugin
      "capabilities": { "portMappings": true }
    }
  ]
}
```

Khi runtime chạy:
- Đầu tiên gọi calico ADD → gán IP cho pod.
- Sau đó gọi portmap ADD → cấu hình port forward (ví dụ port 80 từ host sang pod).

→ Plugin calico có thể delegate (nhường lại) phần port mapping cho plugin portmap → đây là quy trình uỷ quyền cho plugin khác.

5. Định dạng JSON chuẩn cho plugin trả kết quả cho runtime
Sau khi plugin xử lý, nó phải trả về một JSON có dạng chuẩn, ví dụ:
```
{
  "cniVersion": "0.8.0",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "aa:bb:cc:dd:ee:ff"
    }
  ],
  "ips": [
    {
      "version": "4",
      "address": "192.168.0.10/24",
      "gateway": "192.168.0.1"
    }
  ]
}
```
Runtime (containerd) đọc ips và interfaces này để:
- Gắn IP đúng cho interface trong container.
- Thiết lập default gateway, routes.




---
Bạn hoàn toàn có thể viết CNI plugin/custom CNI riêng chứ không bị giới hạn chỉ dùng các plugin phổ biến như Calico, Flannel, Cilium, Amazon VPC CNI, v.v.


Một custom CNI plugin là chương trình (thường là một binary) trên mỗi node thỏa mãn:
- Nhận đúng input (stdin JSON) theo CNI spec:
- ADD, DEL, CHECK command.
- chứa CNI_COMMAND, CNI_CONTAINERID, CNI_NETNS, CNI_IFNAME, CNI_ARGS, CNI_PATH, và cấu hình mạng.
- Trả về đúng output (stdout JSON) theo spec: IP, interface, gateway, routes, DNS… để runtime gắn vào pod.

Về cơ bản, bạn có thể viết bằng bất kỳ ngôn ngữ nào (Go, Bash, Python, Rust…) miễn là nó ra một binary chạy đúng giao thức CNI.

Ví dụ một repo custom-cni mô tả cách viết một CNI plugin đơn giản cho k8s: https://github.com/ronak-agarwal/custom-cni

---

Để chỉ định pod thuộc mạng dbnet (mạng CNI bạn vừa định nghĩa), bạn có 2 trường hợp chính:
- Nếu dbnet là mạng mặc định của cụm → pod tự động thuộc mạng đó.
- Nếu trong cụm có nhiều CNI/mạng → bạn cần dùng Multus + NetworkAttachmentDefinition để chỉ định pod dùng đúng CNI config dbnet.


#### Trường hợp 1: dbnet là mạng CNI chính của cụm
Khi bạn đã đặt file config vào /etc/cni/net.d/xxxx-dbnet.conflist trên tất cả worker node thì mặc định mọi pod sẽ dùng mạng này cho interface chính (eth0).

#### Trường hợp 2: Cụm dùng nhiều mạng → muốn pod dùng dbnet (custom CNI)
Nếu cluster đang dùng một CNI mặc định (ví dụ: Calico/Cilium), mà bạn muốn một số pod (đặc biệt là DB) dùng riêng mạng dbnet, bạn cần:
- Cài Multus CNI - là một “meta CNI plugin” cho phép gắn nhiều interfaces với nhiều mạng khác nhau cho một pod. Multus sẽ đọc các file CNI config (ví dụ: dbnet, default, management) và expose cho Kubernetes dưới dạng NetworkAttachmentDefinition.
- Tạo NetworkAttachmentDefinition cho dbnet
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


Sau đó gắn pod vào mạng dbnet qua annotation

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

Nếu bạn muốn pod chỉ dùng mạng dbnet hoàn toàn, thì bạn phải cấu hình Multus để đặt dbnet làm interface chính (thay đổi cấu hình Multus trên node).

---

### Intra-node Container Networking 
- là giao tiếp giữa các container trong cùng một node. Lúc này Linux kernel sẽ xử lý toàn bộ traffic, không cần đi qua NIC vật lý.

```
[ Container A ]     [ Container B ]
  eth0 (veth0)        eth0 (veth1)
       |                   |
  veth0 <pair>        veth1 <pair>
  ┌────┴───────────────────┴────┐
  │        Linux Bridge         │  ← docker0 / cni0 / cbr0
  │     (Layer 2 switching)     │
  └─────────────────────────────┘
```

- Bridge thực hiện MAC-based forwarding (L2), hoặc kernel routing nếu khác subnet (L3). Ưu điểm là latency rất thấp vì chỉ là memory copy giữa các network namespace.

- Trong Kubernetes: bridge thường là cni0, do CNI plugin tạo ra.

### Inter-node Container Networking
- Là trường hợp các container giao tiếp khác node, phức tạp hơn vì packet phải đi qua mạng vật lý (underlay). Có nhiều cách để giải quyết bài toán này:
  - Overlay Network (VXLAN / Geneve): Dùng bởi Flannel VXLAN, Calico VXLAN, Cilium. Pod IP packet được đóng gói (encapsulate) vào UDP frame trước khi gửi qua mạng vật lý. Ưu điểm: Không yêu cầu cấu hình router/switch phức tạp. Nhược điểm: Overhead encap/decap, MTU phải giảm (~50 bytes cho VXLAN header).
  - Native Routing (BGP / Direct Route): Dùng bởi: Calico BGP, Cilium native routing, Flannel host-gw. Ưu điểm: Performance tốt nhất, không MTU overhead, dễ debug. Nhược điểm: Yêu cầu router/switch hỗ trợ, hoặc tất cả node phải cùng L2 segment (với host-gw).
  - eBPF Datapath: Dùng bởi: Cilium


### Linux bridge 
- là một virtual network switch được implement trong Linux kernel — hoạt động giống hệt một con switch vật lý Layer 2, nhưng hoàn toàn bằng software. Thay vì port vật lý, bridge có các network interface gắn vào làm port.
- Cách hoạt động: Bridge duy trì một Forwarding Database (FDB) — bảng ánh xạ MAC address → port, giống MAC table của switch thật.
- Xem FDB của bridge bằng lênhk `bridge fdb show br docker0`
  ```
  aa:bb:cc:dd:ee:01  dev veth0   # Container A ở port veth0
  aa:bb:cc:dd:ee:02  dev veth1   # Container B ở port veth1
  ```
- Khi container A gửi packet đến container B:
  - Packet đi qua veth pair vào bridge
  - Bridge tra FDB theo MAC đích
  - Forward ra đúng port → vào veth pair của container B. Toàn bộ quá trình xảy ra trong kernel memory, không ra NIC vật lý

- Tạo bridge thủ công
```
ip link add name br0 type bridge
ip link set br0 up
```

- Gắn interface vào bridge (làm "port")
```
ip link set eth1 master br0
ip link set veth0 master br0
```
- Đặt IP cho bridge (để host giao tiếp được)
```
ip addr add 192.168.1.1/24 dev br0
```


- Trong Docker / Kubernetes

| | Docker | Kubernetes |
|---|---|---|
| Bridge name | `docker0` | `cni0` (do CNI tạo) |
| Tạo bởi | Docker daemon | CNI plugin (Flannel, Calico...) |
| Mỗi container có | 1 veth pair | 1 veth pair |

---

Khi 1 pod trong cụm k8s được tạo ra thì interface của nó sẽ được gắn với linux bridge hoặc tùy thuôc CNI plugin. Với đa số CNI phổ biến thì có bridge

Flow khi Pod được tạo (với CNI dùng bridge)
```
kubelet tạo Pod
      │
      ▼
Container Runtime (containerd/cri-o)
tạo network namespace cho Pod
      │
      ▼
kubelet gọi CNI plugin
(ví dụ: flannel, calico, weave)
      │
      ▼
CNI plugin thực hiện:

1. Tạo veth pair
   vethXXXXXX  ◄────────────►  eth0
   (host side)                (pod side, nằm trong pod netns)

2. Gắn vethXXXXXX vào bridge
   cni0 (bridge)
   └── vethXXXXXX  ← pod mới

3. Gán IP cho eth0 của pod
   (lấy từ IPAM - IP Address Management)

4. Cấu hình route trong pod netns
```

Kết quả sau khi pod chạy
- Trên host, thấy bridge và các veth
```
ip link show type bridge
# → cni0

bridge link show
# → vethABC  master cni0    ← pod 1
# → vethDEF  master cni0    ← pod 2
# → vethGHI  master cni0    ← pod mới vừa tạo

# Bên trong pod
kubectl exec -it <pod> -- ip addr
# → eth0: 10.244.1.5/24    ← đầu còn lại của veth pair
```

Lưu ý không phải lúc nào cũng có bridge. Với Calico, thay vì bridge, Calico gắn veth trực tiếp vào **routing table của host**:

| CNI Plugin | Có dùng bridge? | Cơ chế |
|---|---|---|
| Flannel (VXLAN) | ✅ Có | `cni0` bridge + flannel.1 VTEP |
| Flannel (host-gw) | ✅ Có | `cni0` bridge + static route |
| Calico | ❌ Không | Dùng **routing thuần L3**, veth gắn thẳng vào routing table |
| Cilium | ❌ Không | **eBPF**, không cần bridge hay veth truyền thống |
| Weave | ✅ Có | `weave` bridge |

---


```
Pod eth0 ──── vethXXX (host) ──── kernel routing table
                                        │
                              route: 10.244.1.5 → vethXXX
```

Không có bridge ở giữa — packet đến host được route thẳng vào đúng veth của pod. Ít overhead hơn, nhưng cần kernel xử lý L3 routing cho từng pod.

Tóm lại: Với Flannel/Weave thì có bridge (cni0). Với Calico/Cilium thì không có bridge — họ dùng L3 routing hoặc eBPF để hiệu quả hơn.

---

OVS vs Linux Bridge

- OVS là virtual switch Layer 2 giống Linux bridge nhưng nhiều tính năng hơn
- Linux bridge chỉ có kernel module. OVS có 3 thành phần:
```
┌─────────────────────────────────────────────┐
│              Userspace                      │
│                                             │
│  ovs-vswitchd  ←→  ovsdb-server             │
│  (daemon xử lý)     (config database)       │
├─────────────────────────────────────────────┤
│              Kernel Space                   │
│                                             │
│         openvswitch.ko                      │
│     (fast path - datapath)                  │
└─────────────────────────────────────────────┘
```
  - ovsdb-server: lưu config (bridges, ports, flows)
  - ovs-vswitchd: xử lý control plane, cài flow vào kernel
  - openvswitch.ko: fast path, forward packet theo flow table

Packet đầu tiên lên userspace để tạo flow, các packet sau được kernel xử lý trực tiếp theo flow đã cache → gần bằng tốc độ Linux bridge.

- Điểm khác biệt chính giữa OVS và Linux bridge là OVS làm tunnel native, còn Linux bridge cần thêm flannel.1 VTEP bên ngoài để làm VXLAN:
  - Linux bridge cần interface riêng cho tunnel: cni0 (bridge) → flannel.1 (VTEP) → eth0
  - OVS tự làm tunnel ngay trong switch
  ```
  ovs-br0 (OVS bridge)
  ├── veth0  ← pod 1
  ├── veth1  ← pod 2
  └── vxlan0 ← tunnel port (OVS tự encap/decap)
  ```
- Ai dùng OVS trong k8s?
  - OpenStack Neutron            → OVS là default
  - OVN (Open Virtual Network) là lớp abstraction bên trên OVS — nếu OVS là switch, thì OVN là cả một virtual network fabric (router, switch, ACL, load balancer).


---
- Trong openstack, Pod là thành phần nằm bên trong VM. VM do OpenStack quản lý, còn Pod chạy bên trong VM đó — VM đóng vai trò là Kubernetes node.
```
┌──────────────────────────────────────────────┐
│              OpenStack (IaaS)                │
│                                              │
│  ┌──────────────┐    ┌──────────────┐        │
│  │     VM 1     │    │     VM 2     │        │
│  │  (Nova)      │    │  (Nova)      │        │
│  │              │    │              │        │
│  │  ┌────────┐  │    │  ┌────────┐  │        │
│  │  │ Pod A  │  │    │  │ Pod C  │  │        │
│  │  ├────────┤  │    │  ├────────┤  │        │
│  │  │ Pod B  │  │    │  │ Pod D  │  │        │
│  │  └────────┘  │    │  └────────┘  │        │
│  │   k8s node   │    │   k8s node   │        │
│  └──────────────┘    └──────────────┘        │
└──────────────────────────────────────────────┘
```

- Hai lớp networking hoàn toàn tách biệt
```
┌─────────────────────────────────────────────────┐
│  Pod Network (Kubernetes - CNI)                  │
│  10.244.0.0/16                                   │
│                                                  │
│   Pod A (10.244.1.5) ──── Pod B (10.244.1.6)    │
│        │                        │                │
│       eth0                     eth0              │
│   ════════════ cni0 bridge ════════════          │
│        │          (inside VM)                    │
├────────┼─────────────────────────────────────────┤
│  VM Network (OpenStack - Neutron)                │
│  192.168.1.0/24                                  │
│                                                  │
│   VM 1 (192.168.1.10) ──── VM 2 (192.168.1.11)  │
│        │                        │                │
│    tap interface             tap interface        │
│   ════════════ OVS bridge ═════════════          │
│              (on hypervisor)                     │
└─────────────────────────────────────────────────┘
```
  - Neutron quản lý network giữa các VM → VM thấy nhau qua 192.168.x.x
  - CNI plugin quản lý network giữa các Pod → Pod thấy nhau qua 10.244.x.x
  - Pod muốn ra ngoài VM phải đi qua eth0 của VM (gateway của pod network)
  - Khi Pod A (VM1) nói chuyện với Pod B (VM2): Pod traffic phải đi xuyên qua VM network của OpenStack — đây là lý do khi dùng Overlay (VXLAN) trong k8s trên OpenStack sẽ bị double encapsulation: VXLAN của CNI nằm trong VXLAN của Neutron.
  ```
  Pod A (10.244.1.5)
    │  cni0 bridge
    │  eth0 của VM1 (192.168.1.10)   ← lớp Neutron
    │  Neutron router / OVS
    │  eth0 của VM2 (192.168.1.11)
    │  cni0 bridge
  Pod B (10.244.2.7)
  ```

- OVS-CNI Plugin giúp giải quyết vấn đề, pod không cần phải qua cni0 bridge để ra ngoài. Thay vì CNI tạo bridge cni0 rồi mới ra OVS, pod gắn veth trực tiếp vào OVS bridge.
- Cách thực hiện: dùng Multus + OVS-CNI. Multus là CNI cho phép gán nhiều network interfaces cho pod
```
┌─────────────────────────────────────────────┐
│                    VM (k8s node)             │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │              Pod                     │   │
│  │  eth0 (primary)   net1 (secondary)   │   │
10.244.1.5/24          192.168.100.10/24
│  │    │                   │             │   │
│  └────┼───────────────────┼─────────────┘   │
│       │                   │                 │
│   flannel/calico      veth pair             │
│   (normal CNI)             │                │
│                        OVS Bridge           │
│                        (br-int / br-ex)     │
└─────────────────────────────────────────────┘
```

-> Double encapsulation problem giải quyết được
- Trước (double encap) `Pod → cni0 → VXLAN (k8s) → eth0 VM → VXLAN (Neutron) → physical`
- Sau (OVS-CNI, single encap) `Pod → OVS br-int → VXLAN (Neutron only) → physical`. Bỏ hoàn toàn lớp encapsulation của k8s CNI, chỉ còn Neutron lo.


### Một cách khác: SR-IOV

Nếu cần **performance cực cao**, dùng SR-IOV để bypass cả OVS:
```
Pod net1 ── VF (Virtual Function) ── PF (Physical NIC) ── physical
Packet đi thẳng từ pod ra NIC, không qua kernel network stack. Nhưng cần hardware hỗ trợ SR-IOV và phức tạp hơn nhiều.

Tóm lại: Dùng Multus + OVS-CNI là cách phổ biến nhất để pod gắn trực tiếp vào OVS bridge, bỏ qua cni0. Đặc biệt hữu ích khi chạy k8s trên OpenStack để tránh double encapsulation.
