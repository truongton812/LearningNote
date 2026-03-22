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
