Học lại về hạ tầng pki, rất hay

<img width="1024" height="576" alt="image" src="https://github.com/user-attachments/assets/15448756-d5b1-43dd-89ed-4157f4c36ece" />

---

Container under the hood

Container bản chất là sử dụng linux namespace
- pid namespace: để isolate process. Các process không thể nhìn thấy nhau, tuy nhiên bản chất vẫn là những process chạy trên máy host
- mount namespace
- network namespace
- user namespace

Ngoài namespace, container còn sử dụng cgroup để giới hạn tài nguyên 1 container có thể sử dụng

---

Từ sau bản k8s 1.28, dùng lệnh docker ps sẽ ko thấy container, phải dùng lệnh crictl ps

Lưu ý 
- docker là container runtime + tool để manage container và image
- containerd là container runtime
- podman là tool để manage container và image
- crictl là cli cho cri-compatible container runtime. Ta có thể dùng crictl để tương tác với docker/podman/containerd tùy ý, chỉ cần config runtime endpoint trong /etc/crictl.yaml

## 4. Network policy

Network policy là namespaced resource

- Network policy cho phép tất cả Pod trong default namespace  giao tiếp bình thường
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector: {}
  policyTypes: []
```
- Network policy kiểm soát inbound traffic (ingress) của tất cả các Pod trong namespace default. Tuy nhiên, vì trong trường này không có quy tắc cụ thể nào trong phần ingress: (không định nghĩa rule cho phép hoặc từ chối traffic nào), Kubernetes mặc định sẽ chặn toàn bộ lưu lượng vào các Pod này
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```
- Network policy cô lập hoàn toàn các Pod trong default namespace, chặn cả outbound và inbound traffic. Lưu ý trong thực tế ta sẽ dùng default network policy là deny all như thế này, sau đó tạo các allow ingress và egress để đảm bảo bảo mật. Tuy nhiên deny all sẽ chặn cả DNS traffic (port 53) nên nếu muốn sử dụng service name trong cụm thì cần allow port 53
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP

```

- Nếu 1 pod được áp nhiều network policy thì sẽ là union của tất cả các network policy áp lên

---

Ví dụ về network policy
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example
  namespace: default
spec:
  podSelector:
    matchLabels:
      id: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          id: ns1
    ports:
    - protocol: TCP
      port: 80
  - to:
    - podSelector:
        matchLabels:
          id: backend
```

Ở phần egress có 2 cái block , sẽ là logic OR

Trong 1 block có 2 key là `to` và `ports`. sẽ là logic AND

Chỉ define Egress, tức Ingress ko bị limit

Trong Kubernetes, các rule NetworkPolicy được xử lý theo thứ tự và rule đầu tiên đã chặn rồi thì rule sau không có tác dụng.

### 4.1 Network policy trong Cloud

Mặc định khi tạo worker node trên cloud thì các pod trên worker node sẽ có quyền truy cập metadata server. Trong đấy có thể chứa các sensitive data (VD credential cho VM/cloud, provisioning data như kubelet credential)

Có thể dùng network policy để restrict pod có quyền truy cập metadata server

Ví dụ dùng network policy để restrict truy cập đến metadata server 169.254.169.254/32
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloud-metadata-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```

## 5. CIS benchmark

CIS Kubernetes Benchmark là bộ tiêu chuẩn cấu hình bảo mật dành riêng cho Kubernetes, được phát triển bởi Center for Internet Security (CIS), cung cấp các hướng dẫn chi tiết để bảo vệ cụm Kubernetes khỏi lỗ hổng và cấu hình sai.​ Bộ benchmark bao gồm hơn 100 kiểm tra phân loại theo 2 cấp độ: Level 1 (cơ bản, dễ triển khai, tác động thấp đến hoạt động) và Level 2 (nâng cao, cho môi trường bảo mật cao). Nó tập trung vào control plane (API server, etcd), worker nodes (Kubelet), RBAC, Pod Security Standards, network policies, và secret management.​

Lợi ích: Áp dụng CIS giúp tuân thủ quy định (GDPR, PCI-DSS), giảm bề mặt tấn công, và là best practice cho DevOps trên AWS EKS hoặc tự quản lý Kubernetes

Người dùng có thể sử dụng kube-bench (mã nguồn mở) để tự động quét và kiểm tra cụm Kubernetes so với benchmark, báo cáo các vấn đề cần khắc phục. 

Dùng lệnh `docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t docker.io/aquasec/kube-bench:latest --version 1.18` cho nhanh -> get được list các lỗ hổng và dựa vào document để fix

## 6. RBAC

- RBAC (Role-Based Access Control) là cơ chế kiểm soát truy cập dựa trên role trong Kubernetes, cho phép quản trị viên quy định ai (người dùng, nhóm hoặc ServiceAccount) có thể thực hiện hành động gì trên tài nguyên cụ thể, áp dụng nguyên tắc "least privilege" để tăng cường bảo mật.
- Mặc định RBAC được enable Kubernetes từ version 1.6 trở lên
- RBAC có thể kết hợp với các authorizer khác như ABAC nếu cần.
- Mặc định khi RBAC được bật trên Kubernetes thì các ServiceAccount (SA) không có quyền gì cả. Nói cách khác, các SA, người dùng mới sẽ không được cấp quyền truy cập hoặc thao tác trên tài nguyên Kubernetes cho đến khi có Role hoặc ClusterRole được tạo và gán quyền (binding) cho SA hoặc user đó. Ví dụ, để một ServiceAccount mặc định có thể list các Service trong một namespace, ta cần tạo Role và RoleBinding gán quyền rõ ràng cho nó, không có quyền nào được cấp mặc định.​
  
Các thành phần chính của RBAC

Kubernetes định nghĩa bốn đối tượng RBAC chính: Role, ClusterRole, RoleBinding và ClusterRoleBinding.​

- Role: Quy định các quyền (verbs như get, list, create) trên tài nguyên trong một namespace cụ thể, ví dụ cho phép list pods trong namespace default.​

- ClusterRole: Quy định các quyền trên tài nguyên trong tất cả các namespace và tài nguyên thuộc non-namespaced (VD Node, Persistent Volume)

- RoleBinding: Gán Role cho subject (User, Group, ServiceAccount) trong một namespace.​

- ClusterRoleBinding: Gán ClusterRole cho subject toàn cluster.​

Cách thức hoạt động: Khi một yêu cầu API đến (như kubectl get pods), Kubernetes kiểm tra RBAC để xác định quyền dựa trên rules trong Role/ClusterRole và bindings.  


- Lưu ý khi dùng lệnh kubectl get các tài nguyên mà không cần cấu hình RBAC thêm vì file kubeconfig mặc định (thường ở ~/.kube/config hoặc /etc/kubernetes/admin.conf) chứa credentials của tài khoản cluster-admin với quyền cao nhất (ClusterRoleBinding cluster-admin được tạo tự động khi cài cluster). Khi tạo cluster (qua kubeadm, Minikube, managed service như EKS/GKE), Kubernetes tự động sinh kubeconfig admin với certificate hoặc token cấp quyền cluster-admin, cho phép truy cập toàn bộ tài nguyên cluster-scoped và namespace mà không cần RBAC riêng.​

- ClusterRole cluster-admin được bind với Group system:masters hoặc admin user trong kubeconfig, cấp verbs * (tất cả hành động) trên resources * (tất cả tài nguyên), bao gồm get/list pods, nodes, deployments. Kiểm tra quyền bằng lệnh: `kubectl auth can-i '*' '*' --all-namespaces` , kết quả "yes" xác nhận quyền admin đầy đủ từ kubeconfig hiện tại.​

- Ngược lại, các ServiceAccount default (như default trong namespace) không có quyền gì, cần RoleBinding riêng để kubectl (qua SA) hoạt động; kubeconfig admin dùng user credentials riêng biệt. Nếu chia sẻ cluster, tạo kubeconfig riêng với RBAC hạn chế thay vì dùng admin

### 6.1 Namespaced và non-namespaced resource
- Xem bằng lệnh `kubectl api-resources --namespaced=true/false`


## 11. RBCA

- Service Account trong Kubernetes là một “danh tính” dành cho workload (Pod, controller, addon…), không phải cho người dùng đăng nhập trực tiếp; Pod dùng Service Account để xác thực với API server và được gán quyền thông qua RBAC (Role/ClusterRole + RoleBinding/ClusterRoleBinding). Service Account là resource nằm trong namespace, mỗi namespace khi tạo ra sẽ có sẵn một ServiceAccount tên `default`, và khi tạo thêm SA thì controller của cluster sẽ tạo object ServiceAccount tương ứng trong etcd.​
- Khi một Pod gọi API bằng token của Service Account, API server sẽ map token đó thành một username nội bộ có dạng system:serviceaccount:<namespace>:<serviceaccount-name>.​ Username kiểu system:serviceaccount:ns:name: do API server tự “build” từ SA + token, không phải user quản trị tạo bằng lệnh kubectl create user.​

### 11.0. ServiceAccount và Normal User

ServiceAccount (SA) trong Kubernetes là một tài nguyên namespaced, được quảng lý bởi k8s API, được sử dụng để đại diện cho danh tính của các tiến trình chạy trong Pod, giúp Pod xác thực với API server mà không cần quản lý credentials thủ công. Token của SA được tự động mount vào container tại /var/run/secrets/kubernetes.io/serviceaccount, cho phép ứng dụng bên trong Pod thực hiện các yêu cầu API với username dạng system:serviceaccount:<namespace>:<sa-name>.​

Normal User (hay còn gọi là Humans/Users) đại diện cho người dùng bên ngoài cluster, không phải là tài nguyên trong k8s, như quản trị viên hoặc developer sử dụng kubectl, được xác thực qua certificates, tokens, hoặc OpenID Connect mà không được quản lý tự động bởi Kubernetes (VD được quản lý bởi AWS, GCP,...). Chúng thuộc các group mặc định như system:authenticated và được phân quyền qua RBAC (Role/ClusterRole với RoleBinding/ClusterRoleBinding), khác với SA chỉ giới hạn trong namespace

Normal User chỉ cần có certificate và key thì sẽ connect được tới cụm k8s. Với điều kiện là certificate của user (hay còn gọi là client certificate) phải được signed bởi cluster's certificate authority (CA). Username trong Kubernetes được định nghĩa trong certificate thông qua trường Common Name (CN) trong phần subject của X.509 client certificate, ví dụ /CN=username sẽ được dùng làm tên người dùng khi xác thực với API server.

Quy trình để tạo client certificate. Ta hoàn toàn có thể tạo CSR rồi dùng CA để ký, sau đó down CRT về mà không cần phải tạo CertificateSigningRequest resource, tuy nhiên như thế sẽ phức tạp hơn

<img width="1307" height="636" alt="image" src="https://github.com/user-attachments/assets/efcbc22a-fc15-41fe-b977-37a4a10aabba" />


### 11.1. Role và clusterrole

- Role là tài nguyên namespace-scoped, định nghĩa quyền truy cập chỉ trong một namespace cụ thể.​
  - Có thể có tạo nhiều role với cùng tên, chỉ cần chúng khác namespace
  - User X có thể gán với nhiều role trên nhiều namespace. VD user X có thể có quyền đọc secret trong namespace1, có quyền đọc/ghi secret trong namespace2
- ClusterRole là tài nguyên cluster-scoped, không thuộc namespace nào và có thể định nghĩa quyền truy cập trên toàn bộ cluster, bao gồm cả tài nguyên có scope namespace và tài nguyên có scope cluster như node hoặc persistent volume.​ ClusterRole thường dùng để tái sử dụng quyền chung trên nhiều namespace hoặc cho tài nguyên không thuộc namespace.​ Khi 1 user được gán với ClusterRole thì user đấy sẽ có quyền giống nhau trên toàn bộ namespace. VD tạo ClusterRole với quyền get secret rồi gán cho user ⭢ user đó có quyền get secret trên toàn bộ namespace
  - Lưu ý ClusterRole sẽ áp dụng lên tất cả namespace hiện có và **các namespace trong tương lai**
 
### 11.2 Kết hợp giữa Role/ClusterRole và RoleBinding/ClusterRoleBinding
Có thể có các tổ hợp

#### 11.2.1 Role + RoleBinding
<img width="1649" height="817" alt="image" src="https://github.com/user-attachments/assets/8a1c0be1-aee6-4454-a482-c295fc7a28c4" />

Kết quả: User có quyền trên 1 namespace cụ thể

#### 11.2.2 ClusterRole + ClusterRoleBinding
<img width="1224" height="571" alt="image" src="https://github.com/user-attachments/assets/c5ffa2a2-5a09-4f5e-8c99-21f41a3f1e2d" />

Kết quả: User có quyền trên toàn bộ namespace và các non-namespaced resources

#### 11.2.3 ClusterRole + RoleBinding
<img width="1201" height="566" alt="image" src="https://github.com/user-attachments/assets/d2ccb287-ebb3-4e89-ae38-8c470ae0350b" />

Kết quả: User có quyền trên 1 số namespace được chỉ định

#### 11.2.3 Role + ClusterRoleBinding
<img width="1091" height="568" alt="image" src="https://github.com/user-attachments/assets/a6164a87-503b-48ca-b544-6ae4b1887b09" />

Không thể thực hiện được

Dùng lệnh `kubectl auth can-i` để test . VD `kubectl auth can-i -n <namespace> get secrets --as <service_account/user>` (hoặc thay vì chỉ định cụ thể 1 namespace thì dùng -A để chỉ định tất cả ns) -> kết quả trả về sẽ là yes hoặc no

### 11.2. RoleBinding và ClusterRoleBinding
- RoleBinding gán quyền từ Role (hoặc ClusterRole) cho user/group/ServiceAccount chỉ trong namespace của nó.​ RoleBinding có thể tham chiếu ClusterRole để áp dụng quyền cluster-wide nhưng giới hạn trong namespace của binding.​
- ClusterRoleBinding gán quyền từ ClusterRole cho subject trên toàn cluster, áp dụng cho mọi namespace.​ Có thể dùng ClusterRoleBinding để bind ClusterRole với ServiceAccount, ví dụ: `kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default.`. Điều này cấp quyền cluster-wide cho ServiceAccount, cho phép truy cập tài nguyên ở mọi namespace hoặc cluster-scoped resources.​ ClusterRoleBinding chỉ mở rộng phạm vi namespace-scoped (đọc mọi namespace)

So với RoleBinding bind cùng ClusterRole, ClusterRoleBinding khác ở phạm vi: RoleBinding chỉ giới hạn namespace của binding, còn ClusterRoleBinding áp dụng toàn cluster.


Ví dụ cụ thể với ClusterRole "view"

- Giả sử có ClusterRole mặc định tên "view" cho phép đọc (get, list, watch) các tài nguyên namespace như Pod, Service, ConfigMap trên toàn cluster.​
- Tạo ServiceAccount "myapp" trong namespace "dev".​ Pod dùng SA này sẽ có quyền khác nhau tùy binding.

Trường hợp dùng RoleBinding với ClusterRole "view": Tạo RoleBinding trong namespace "dev":
`kubectl create rolebinding myapp-view-dev --clusterrole=view --serviceaccount=dev:myapp --namespace=dev`

-> SA "myapp" chỉ đọc được tài nguyên trong namespace "dev" (như kubectl get pods -n dev).​ Không thể đọc Pod ở namespace khác như "prod" (kubectl get pods -n prod sẽ lỗi).​

Trường hợp dùng ClusterRoleBinding với ClusterRole "view". Tạo ClusterRoleBinding:
`kubectl create clusterrolebinding myapp-view-cluster --clusterrole=view --serviceaccount=dev:myapp`

-> SA "myapp" đọc được tài nguyên namespace-scoped ở mọi namespace (như kubectl get pods -n dev hoặc kubectl get pods -n prod).​ Vẫn không đọc cluster-scoped resources như Node do ClusterRole "view" mặc định chỉ định nghĩa quyền đọc (get, list, watch) cho namespace-scoped resources như Pod, Service, ConfigMap, Secret, nó không bao gồm cluster-scoped resources như Node, PersistentVolume, Namespace. Muốn đọc được cluster-scoped resources thì cần tạo ClusterRole cho phép đọc cluster-scoped resources (ví dụ ClusterRole "system:node-reader" cho phép list Node), sau đó bind bằng ClusterRoleBinding với ServiceAccount hoặc User (không dùng RoleBinding vì nó là namespace-scoped)
