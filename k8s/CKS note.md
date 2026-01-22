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
- Trong Kubernetes, mặc định mọi Pod đều có thể truy cập mọi Pod
- Để kiểm soát traffic giữa các Pod trong cluster thì cần sử dụng Network Policy. Network policy trong Kubernetes là một resource gắn với namespace, dùng để định nghĩa các quy tắc kiểm soát traffic, hoạt động như một firewall ở tầng IP/port (OSI layer 3–4) cho các Pod.
- Network Policy dùng label để chọn nhóm Pod chịu tác động (qua podSelector), và định nghĩa các rule ingress (traffic vào) và egress (traffic ra) nào được phép.
- Khi trong trường `policyTypes` của Network policy có `Egress` hay `Ingress` thì pod bị áp policy ngay lập tức sẽ bị isolated (deny traffic). Nếu muốn pod có thể gửi/nhận request thì cần allow cụ thể (đã kiểm nghiệm thực tế không cần khai báo `policyTypes` mà chỉ cần khai báo egress/ingress list là Pod đã bị chặn rồi, tuy nhiên nên khai báo cho dễ đọc)
- Khi đã allow 1 chiều thì chiều còn lại sẽ được hiểu rằng cũng allow (stateless)
- Nếu 1 Pod được áp nhiều network policy thì sẽ là union của tất cả các network policy áp lên
- Trong 1 Network Policy, các rule được xử lý theo thứ tự và rule đầu tiên đã chặn rồi thì rule sau không có tác dụng.
- Lưu ý Policy chỉ có hiệu lực nếu CNI (ví dụ Calico, Cilium, Weave Net) hỗ trợ NetworkPolicy; nếu không, policy chỉ là một object trong API server chứ không kiểm soát được traffic (VD Flannel)

### 4.0. Logic giữa các thành phần trong rule
Trong cùng 1 block thì logic giữa các rule là AND, khác block thì là logic OR. Ví dụ
<img width="896" height="406" alt="image" src="https://github.com/user-attachments/assets/20247a08-fe9e-492e-b809-011ebfb6228f" />

Chỉ cần thay đổi block thì logic cũng thay đổi theo
<img width="739" height="402" alt="image" src="https://github.com/user-attachments/assets/46bbffa8-7be9-47f6-8e88-59ea0cc36540" />

### 4.1. Network policy cho phép tất cả Pod trong default namespace giao tiếp bình thường (mặc định)
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

### 4.2. Network policy chặn inbound vào Pod
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

### 4.3. Network policy chặn inbound và ountbound traffic (cô lập hoàn toàn), chỉ cho phép TCP/UDP port 53
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
- Trong thực tế production sẽ dùng default network policy là deny all, sau đó tạo các allow ingress và egress để đảm bảo bảo mật.
- Cần allow DNS traffic (port 53) để có thể sử dụng Service name

### 4.4. Network policy cụ thể
- Network policy này áp lên pod có label là `id=frontend` trong namespace default, không kiểm soát inbound traffic (do không define Ingress), chỉ kiểm soát outbound traffic theo rule:
  - cho phép kết nối đến port 80 của các Pod ở namespace có label `id=ns1` (lưu ý là chỉ cho phép khi **destination port** = 80/TCP trên Pod đích, còn source port của Pod frontend có thể là bất kỳ port nào. Ví dụ Pod frontend gọi curl http://backend-ns1-service:80 → Được phép)
  - cho phép kết nối đến tất cả các Pod có label `id=backend` ở namespace `default`
- Lưu ý: ở phần egress có 2 block , sẽ là logic OR. Còn trong 1 block có 2 key là `to` và `ports`, sẽ là logic AND

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
### 4.5. Network policy chỉ cho phép Pod giao tiếp với các Pod trong cùng namespace
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
  namespace: mynamespace
spec:
  podSelector:
    matchLabels:
      app: myapp
  egress:
    - to: #trường này để cho phép pod gọi đến coredns
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
    - to: #trường này để cho phép pod giao tiếp trong cùng namespace
        - podSelector: {}
```

Lưu ý: Do NetworkPolicy là namespace-scoped resource → rule `ingress.from.podSelector: {}` và `egress.to.podSelector: {}` chỉ match các pod trong `mynamespace`, do đó không cần chỉ định namespace nữa

### 4.6. Network policy trong Cloud

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

ServiceAccount (SA) trong Kubernetes là một tài nguyên namespaced, được quảng lý bởi k8s API, được sử dụng để đại diện cho danh tính của các tiến trình chạy trong Pod, giúp Pod xác thực với API server mà không cần quản lý credentials thủ công. Khi tạo SA account resource, thực chất k8s sẽ tạo ra 1 secret chứa token. Token của SA được tự động mount vào container tại /var/run/secrets/kubernetes.io/serviceaccount (hoặc dùng lệnh `mount | grep serviceaccount` để tìm vị trí mount), cho phép ứng dụng bên trong Pod thực hiện các yêu cầu API với username dạng system:serviceaccount:<namespace>:<sa-name>.​

Trong thư mục /var/run/secrets/kubernetes.io/serviceaccount sẽ có các file ca.crt, namespace và token

Có thể sử dụng token tại /var/run/secrets/kubernetes.io/serviceaccount/token để gọi đến api server bằng lệnh `curl https://10.96.0.1 -k -H 'Authorization: Bearer $(cat token)` -> lúc này ta có thể tương tác với api server bằng service account tương ứng với token (lưu ý mới pass qua được bước authen, còn để thực hiện action thì phải xem service account có quyền gì)


Lệnh tạo service account `kubectl create serviceaccount <name>`

Lệnh cấp quyền cho service account : dùng rolebinding

Lệnh để lấy token của service account: `kubectl create token <sa_name>` (lưu ý chỉ generate được temp token, có thể dùng jwt decoder để đọc thông tin) (hoặc đọc secret rồi decode base64 - cần check lại thông tin)

---

automountServiceAccountToken dùng để bật/tắt việc tự động mount token của ServiceAccount vào Pod, từ đó quyết định Pod có sẵn cred để gọi Kubernetes API hay không.​

Mặc định, khi không cấu hình gì, Kubernetes sẽ tự gắn ServiceAccount (thường là default) cho Pod và tự động mount token vào thư mục như /var/run/secrets/kubernetes.io/serviceaccount/token để Pod có thể auth tới API server.​

Điều này tiện cho app nào cần gọi K8s API, nhưng cũng làm tăng surface tấn công nếu container bị compromise vì attacker có thể dùng token đó.​

Khi đặt automountServiceAccountToken: false ở level ServiceAccount hoặc Pod, Kubernetes sẽ không mount token vào Pod nữa, Pod sẽ không có sẵn cred để gọi API server qua token đó.​

Cấu hình có 2 chỗ:
-Trong ServiceAccount YAML: áp dụng cho tất cả Pod dùng SA đó (trừ khi Pod override).​
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```
- Trong Pod.spec: override giá trị trên ServiceAccount, Pod spec luôn được ưu tiên.​
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
...

```
Use case thực tế
- Pod không cần gọi Kubernetes API (app thuần business logic, job đơn giản, sidecar không quản lý cluster) nên disable để giảm quyền và giảm rủi ro nếu bị lộ token.​
- Môi trường security-sensitive (production, multi-tenant, PCI, v.v.): best practice là set automountServiceAccountToken: false cho default SA và chỉ bật cho những Pod/SA thực sự cần gọi API, thậm chí kết hợp với RBAC tối thiểu (least privilege).

---

Normal User (hay còn gọi là Humans/Users) đại diện cho người dùng bên ngoài cluster, không phải là tài nguyên trong k8s, như quản trị viên hoặc developer sử dụng kubectl, được xác thực qua certificates, tokens, hoặc OpenID Connect mà không được quản lý tự động bởi Kubernetes (VD được quản lý bởi AWS, GCP,...). Chúng thuộc các group mặc định như system:authenticated và được phân quyền qua RBAC (Role/ClusterRole với RoleBinding/ClusterRoleBinding), khác với SA chỉ giới hạn trong namespace

Normal User chỉ cần có certificate và key thì sẽ connect được tới cụm k8s. Với điều kiện là certificate của user (hay còn gọi là client certificate) phải được signed bởi cluster's certificate authority (CA). Username trong Kubernetes được định nghĩa trong certificate thông qua trường Common Name (CN) trong phần subject của X.509 client certificate, ví dụ /CN=username sẽ được dùng làm tên người dùng khi xác thực với API server.

Quy trình để tạo client certificate. Ta hoàn toàn có thể tạo CSR rồi dùng CA để ký, sau đó down CRT về mà không cần phải tạo CertificateSigningRequest resource, tuy nhiên như thế sẽ phức tạp hơn

<img width="1307" height="636" alt="image" src="https://github.com/user-attachments/assets/efcbc22a-fc15-41fe-b977-37a4a10aabba" />

Quy trình: 
- User tạo key (có thể dùng openssl)
- từ key user tạo CSR
- Admin tạo CertificateSigningRequest resource trong cụm k8s bằng thông tin CSR user gửi (lưu ý cần mã hóa base64), sau đó admin approve bằng lệnh `kubectl certificate approve <ten_csr>`
- User download cert và thêm vào trong kubeconfig để sử dụng
- Các lệnh làm việc với kubeconfig
  - `kubectl config view`. Thêm option `--raw` để xem data
  - `kubectl config set-credentials <user_name> --client-key=<key> --client-certificate=<cert>`. Thêm option `--embed-certs` để include vào
  - `kubectl config set-context <context_name> --user=<user_name> --cluster=<cluster_name>`

Lưu ý không thể invaliadte 1 certificate
⭢ trong trường hợp certificate bị leak thì xử lý bằng cách:
- Remove tất cả access bằng RBAC
- Tạo CA mới và issue lại tất cả các cert

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

VD: `kubectl auth can-i delete secrets --as system:serviceaccount:default:accessor`

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

Lưu ý permission là additive

<img width="1176" height="397" alt="image" src="https://github.com/user-attachments/assets/bf45b00d-7a61-47a3-8e7d-58ab2a04e270" />



## 12. Workflow request đi tới API server

Khi request đi đến apiserver cần đi qua 3 bước
- authentication
- authorization
- admission control (check lại xem bước này làm gì, trong bài giảng ghi là "has the limit of pods been reached"

<img width="1529" height="665" alt="image" src="https://github.com/user-attachments/assets/9cbf58dd-b787-46c3-8630-e0db6e397cdf" />

1 API request luôn phải tie với normal user/ serviceaccount, nếu không sẽ bị treated là anonymous request.

Tất cả các request đều cần được authenticated, nếu không sẽ bị treated là anonymouse user

Các best pratice về bảo mật
- Không cho phép truy cập anonymous bằng cách sửa file /etc/kubernetes/manifest/kube-apiserver.yaml, thêm option --anonymous-auth=false
- Đóng insecure port bằng cách sửa file /etc/kubernetes/manifest/kube-apiserver.yaml, thêm option --insecure-port=0 (nếu thay bằng port khác thì sẽ là mở insecure port, rất nguy hiểm vì khi đấy truy cập cụm k8s thông qua insecure port sẽ không có cơ chế authentication và authorization, chỉ có admission control. Chỉ phù hợp cho test/debug)
- Không expose API server ra outside
- Restrict access từ node đến apiserver bằng NodeRestriction
- prevent unauthorized access bằng RBAC
- ngăn pod truy cập apiserver
- apiserver port nên nằm sau firewall/ chỉ cho phép truy cập từ 1 ip range cụ thể



Giải thích về các option khi gọi đến apiserver
- gọi không có option gì (VD curl https://10.154.0.2:6443) -> lỗi không có local issuer certificate, không thể thiết lập secure connection
- Nếu curl với option --cacert và trỏ về ca (VD curl https://10.154.0.2:6443 --cacert ca.crt) -> apiserver sẽ nhận dạng đây là anonymous request, tuy nhiên sẽ không có quyền gì. (cảm thấy option --cacert và option -k đều cho ra cùng 1 kết quả, cần check lại)
(nguyên nhân là do từ k8s 1.6 trở lên, anonymous access mặc định được enable. Nếu client gọi API server mà không gửi bất kỳ credential nào (không cert, không token, không basic auth), API server sẽ gán request đó thành “anonymous user” thay vì reject ngay.​ Điều kiện là API server đang bật một mode authorization thực sự (RBAC, ABAC, Node, Webhook, v.v.), không phải mode “AlwaysAllow”.​ Lưu ý thêm là khi sử dụng RBAC/ABAC, user system:anonymous (hoặc group system:unauthenticated) mặc định không có quyền gì; muốn cho anonymous làm gì phải tạo rule/RoleBinding/Policy rõ ràng cho nó.​ Nếu không cấu hình gì cho anonymous, mọi request không auth sẽ bị từ chối bởi layer authorization, dù anonymous access đang “enabled by default”)
- Nếu curl với option --cacert --cert --key (VD curl https://10.154.0.2:6443 --cacert ca.crt --cert cert.crt --key key.pem) -> sẽ có thể truy cập api server

### 12.1 Cách truy cập API server

Có thể gọi đến api server từ các nguồn: outside, pod, node (còn các thành phần của cụm thì hiển nhiên)

Trong k8s có 1 service resource tên kubernetes ở namespace default. Ta có thể gọi đến pod apiserver thông qua service này. Nếu muốn expose ra thì chuyển service kubernetes đấy thành NodePort

Để truy cập API server từ bên ngoài cụm, ta cần sửa file kubeconfig, lưu ý trong apiserver.cert có cấu hình để chỉ allow traffic từ 1 số IP và DNS cụ thể (xem bằng lệnh `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text` trên master node, tìm đến trường `X509v3 Subject Alternative Name`), do đó cần sửa file /etc/hosts để trỏ dns name kubernetes về IP public của cluster (nhớ kèm theo port)), sau đó sửa lại file kubeconfig trỏ về server k8s theo dns name kubernetes)

### 12.2 Cách truy cập API server từ worker node

Trên worker node cũng có cert để gọi đến api server, xem ở cấu hình của kubelet tại /etc/kubernetes/kubelet.conf (file này có vẻ giống file kubeconfig, cũng có cluster/user/context)

Dùng file /etc/kubernetes/kubelet.conf (export vào biến KUBECONFIG) để gọi đến api server thì sẽ có username là `system:node:<node_name>`. Mặc định thì user này chỉ có các quyền là `get node`, đánh label cho chính node đấy (không thể đánh label cho node khác). Lưu ý không đánh được label có string `node-restriction.kubernetes.io` -> đây gọi là noderestriction, viết kết hợp lại với 12.3

### 12.3 mối liên hệ giữa noderestriction và admission controller

- NodeRestriction là một admission controller validating trong Kubernetes, được kích hoạt qua cờ --enable-admission-plugins=NodeRestriction trên kube-apiserver. Nó hạn chế kubelet (thuộc nhóm system:nodes với username system:node:<NodeName>) chỉ sửa đổi Node API object của chính mình và Pod API objects được bind vào node đó. Admission controller ngăn kubelet xóa Node object hoặc thay đổi nhãn có prefix kubernetes.io/ hoặc k8s.io/ một cách tùy ý.​
- NodeRestriction hạn chế Node label mà kubelet có thể modify - chỉ có thể modify label của node đấy và pod trên node đấy, ko modify được node khác (???)

- NodeRestriction chính là một plugin cụ thể thuộc hệ thống admission controllers của Kubernetes, hoạt động ở giai đoạn validating để kiểm tra và từ chối các yêu cầu API không hợp lệ trước khi persist vào etcd. Admission controllers chạy theo thứ tự: mutating trước, validating sau (NodeRestriction thuộc validating), và được enable qua danh sách comma-separated trong kube-apiserver. Mối liên hệ trực tiếp nằm ở việc NodeRestriction được thiết kế dành riêng để bảo vệ Node/Pod objects khỏi các thay đổi không mong muốn từ kubelet, tăng cường bảo mật cluster.​

- Để kích hoạt, chỉnh sửa manifest /etc/kubernetes/manifests/kube-apiserver.yaml trên control plane, thêm NodeRestriction vào --enable-admission-plugins, sau đó kubelet cần dùng credentials phù hợp. Ví dụ: --enable-admission-plugins=NamespaceLifecycle,LimitRanger,NodeRestriction. Kiểm tra bằng kubectl get nodes hoặc logs apiserver sau khi restart.

## 13. Kubernetes version

Kubernetes version có dạng X.X.X (major-minor-patch). Trong đó minor được publish mỗi 3 tháng. K8s version không có LTS

K8s chỉ hỗ trợ support cho 3 bản minor gần nhất

Quy tắc:

- kube-apiserver: Trong cụm HA (high-availability), các instance kube-apiserver mới nhất và cũ nhất phải nằm trong một minor version hoặc lệch nhau tối đa một minor version,  Ví dụ, nếu instance mới nhất chạy phiên bản 1.34, thì các instance khác chỉ được phép ở 1.34 hoặc 1.33, không được xuống 1.32 hoặc thấp hơn
- Các thành phần Kube-controller-manager, kube-scheduler và cloud-controller-manager không được mới hơn kube-apiserver và cũ hơn tối đa 1 minor version.
- kubelet và kube-proxy không được mới hơn kube-apiserver và có thể cũ hơn tối đa 2 minor versions. ví dụ, nếu apiserver ở 1.34 thì kubelet và kube-proxy hỗ trợ 1.34, 1.33. Trước khi nâng cấp kubelet minor version, cần drain pod khỏi node vì không hỗ trợ in-place upgrade.​
- Kubectl hỗ trợ lệch tối đa 1 minor version so với apiserver (có thể lớn hơn). VD apiserver ở 1.33 thì kubectl hỗ trợ 1.34, 1.33 và 1.32. Trong cụm HA với kube-apiserver lệch phiên bản (ví dụ: instance 1.33 và 1.34), kubectl chỉ hỗ trợ đúng các phiên bản đó (1.33 và 1.34), loại trừ 1.35 vì lệch quá xa so với instance cũ nhất. Quy tắc này giúp duy trì ổn định khi nâng cấp dần từng thành phần control plane.

Tóm lại quy tắc quan trọng: trong cụm k8s, các thành phần phải ngang hoặc nhỏ hơn apiserver 1 minor version (trừ kubelet,kube-proxy và kubectl)

Lưu ý Kubernetes Version Skew Policy không có ràng buộc cụ thể riêng biệt cho patch versions giữa các thành phần như kube-apiserver, kubelet hay kube-proxy. Chính sách chỉ quy định độ lệch dựa trên minor versions (ví dụ: kubelet cũ hơn tối đa 3 minor so với apiserver), trong khi patch versions (z trong x.y.z) được khuyến nghị cập nhật lên bản mới nhất trước khi nâng cấp minor để tận dụng các bản sửa lỗi và bảo mật

### 13.1 Upgrade cụm k8s
Các bước cần thực hiện
- Drain node bằng lệnh `kubectl drain`
- Mark node as SchedulingDisabled bằng lệnh `kubectl cordon` (thực chất drain đã bao gồm cordon)
- Upgrade
- Uncordon

Thực hiện trên master node trước (apiserver, controller manager, scheduler). Sau đó thực hiện trên worker node (kubelet, kube-proxy)

Các bước dưới còn thiếu bước update repo, tìm lại trong vở

#### 13.1.1 Upgrade master node
- kubectl drain <node> --ignore-daemonsets
- apt-get update
- apt-cache show kubeadm | grep <version> -> tìm các phiên bản kubeadm khả dụng
- apt-mark hold kubectl kubelet -> để tránh lỗi
- apt-mark unhold kubeadm -> trong TH kubeadm bị hold
- apt-get install kubeadm=<version>. Check lại version bằng kubeadm version
- kubeadm upgrade plan
- kubeadm upgrade apply <version> -> upgrade kubeadm sẽ upgrade luôn các components, xem chi tiết ở lệnh plan
- apt-mark unhold kubectl kubelet
- systemctl restart kubelet
- apt-get install kubelet=<version> kubectl=<version> : do kubelet và kubectl không được update = kubeadm
- kubectl uncordon <node>

#### 13.1.2 Upgrade worker node
- Đứng trên master node để drain/cordon worker node
- Đứng trên worker node chạy các lệnh
- apt-get update
- apt-cache show kubeadm | grep <version>
- apt-mark hold kubelet kubectl
- apt-mark unhold kubeadm
- apt-get install kubeadm=<version>
- kubeadm upgrade node
- hold lại kubeadm
- unhold kubelet và kubectl
- apt-get install kubelet=<version> kubectl=<version>
- hold lại kubelet và kubectl
- restart kubelet
- uncordon node (đứng trên master node)


### 13.2 Cách giúp ứng dụng của bạn “sống sót” khi nâng cấp cluster Kubernetes hoặc hạ tầng bên dưới.​

- Pod terminationGracePeriodSeconds / trạng thái Terminating: cho pod thời gian shutdown sạch sẽ trước khi bị kill.
- Pod Lifecycle Events: các hook như preStop để chạy logic dọn dẹp / drain kết nối khi pod chuẩn bị bị terminate.
- PodDisruptionBudget (PDB): giới hạn số pod có thể bị down cùng lúc khi nâng cấp / bảo trì, giúp tránh downtime toàn bộ ứng dụng.

## 14. Secret trong k8s
- container sẽ thông qua kubelet <-> apiserver <-> etcd để đọc secret lưu trong etcd
- Về cơ bản nếu ta có quyền root để exec vào container trên worker node thì sẽ lấy được thông tin secret (VD dùng lệnh inspect container, đọc đến env hoặc mount path. Với mount volume thì tìm phức tạp hơn chút, phải tìm đến pid của container trên host (cùng dùng lệnh inspect tìm pid), sau đó dùng các lệnh như `ls /proc/<pid>/root` `find /proc/<pid>/root/etc/<secret_name` để tìm và đọc secret. Vì bản chất là secret này được lưu trên host, sau đó mount vào container thôi)

### 14.1 Đọc secret trong etcd
Giả sử hacker có quyền truy cập etcd thì sẽ đọc được secret do thông tin lưu trong etcd mặc định không được mã hóa

Cách truy cập etcd bằng cert và key (lấy từ /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd)

`ETCDCTL_API=3 etcdctl --cert <path> --key <path>  --cacert <path> endpoint health (nếu dùng external etcd thì cần chỉ định --endpoint)`

Cách đọc secret trong etcd

`ETCDCTL_API=3 etcdctl --cert <path> --key <path>  --cacert <path> get /registry/secrets/<namespace>/<secret_name>` -> lấy được secret ở dạng plain text

Để mã hóa secret lưu trong etcd, cần sử dụng EncryptionConfiguration resource (tạo trong /etc/kubernetes/etcd/ec.yaml - không biết tạo chỗ khác được không hay phải tạo ở đây), sau đó khai báo trong file config ở trường --encryption-provider-config

```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - identity: {}
  - aescbc:
      keys:
      - name: key1
        secret: <base64-key> #dungf1 chuỗi túy ý sau đó mã hóa bằng base64 lệnh là echo -n <string> | base64. -n là để bỏ xuống dòng
      - name: key2
        secret: <base64-key>
  - aesgcm:
      keys:
      - name: key1
        secret: <base64-key>
      - name: key2
        secret: <base64-key>
```

-> file này trường provider sẽ đọc từ trên xuống theo thứ tự, provider đầu tiên sẽ được sử dụng để encrypt data và lưu vào etcd. Như hình trên là không có mã hóa do provider đầu tiên là identity. Nếu muốn có mã hóa thì đưa identity xuống dưới. Sau đó nếu muốn thay thế tất cả các secret hiện tại thành mã hóa thì dùng lệnh `kubectl get secrets --all-namespaces -ojson | kubectl replace -f -` (do etcd chỉ sử dụng cơ chế mã hóa cho các secret về sau sau khi ta apply EncryptionConfiguration resource, còn các secret đã tồn tại thì dùng config trước)

Để decrypt tất cả các secret thì lại đảo provider identity lên đầu rồi dùng lệnh `kubectl get secrets --all-namespaces -ojson | kubectl replace -f -`


Sau khi tạo file EncryptionConfiguration trong /etc/kubernetes/etcd/ec.yaml cần sửa config của kube-apiserver, thêm option `--encryption-provider-config=trong /etc/kubernetes/etcd/ec.yaml`

Sửa cả volume thêm cụm hostPath
```
- hostPath:
    path: /etc/kubernetes/etcd
    type: DirectoryOrCreate
  name: etcd
```

Rồi mount vào container
```
- mountPath: /etc/kubernetes/etcd
  name: etcd
```

Lưu ý EncryptionConfiguration chỉ giúp mã hóa secret khi ta đọc trực tiếp tới etcd, còn nếu thông qua apiserver thì vẫn chỉ là base64 như thường, dễ dàng decode (cần check lại thông tin)


EncryptionConfiguration là một tài nguyên cấu hình YAML dùng cho kube-apiserver trong Kubernetes, giúp mã hóa dữ liệu nhạy cảm (như Secrets, ConfigMaps) lưu trữ tại rest trong etcd. Nó được chỉ định qua flag --encryption-provider-config khi khởi động API server, đảm bảo dữ liệu chỉ được mã hóa khi ghi vào etcd và giải mã khi đọc


## 14. Bảo mật container runtime

Về cơ bản container chỉ là các process được cô lập bằng namespaces/cgroups, nhưng bản chất các process này vẫn chạy trên Linux kernel của host và có thể thực hiện system call trực tiếp xuống kernel giống như process bình thường

<img width="1065" height="530" alt="image" src="https://github.com/user-attachments/assets/db8642c8-fa18-4d2f-a9f7-f4a43638576d" />


System call có thể coi như là API mà kernal cung cấp để process có thể giao tiếp với kernal

Để bảo vệ kernal có 2 hướng:
### Hướng 1: Cô lập môi trường thực thi
- Cô lập môi trường thực thi bằng cách thêm 1 lớp sandbox ở giữa. Sandbox là một môi trường cô lập (hay "hộp cát") được sử dụng để chạy phần mềm, ứng dụng hoặc mã đáng ngờ mà không ảnh hưởng đến hệ thống chính, thường áp dụng trong bảo mật máy tính. Sanbox trong trường hợp này có thể xem là 1 lớp bảo mật để hạn chế tấn công. sandbox container runtime cung cấp lớp cách ly bảo mật cao hơn so với runc truyền thống, ngăn chặn container truy cập trực tiếp kernel host để giảm rủi ro escape
<img width="1135" height="488" alt="image" src="https://github.com/user-attachments/assets/fbf25a53-32fd-4731-bafe-db6b8678628f" />

- Khi sử dụng sandbox, container process không thể gọi trực tiếp syscall mà phải thông qua lớp sandbox -> ta có thể kiểm soát syscall thông qua lớp sandbox
- Sử dụng sandbox ta phải đánh đổi:
  - Tốn nhiều resource hơn
  - không phù hợp với các container cần gọi nhiều syscall
  - không còn có thể trực tiếp truy cập hardware do lớp sandbox đã chặn

Kubelet tại 1 thời điểm chỉ có thể chạy 1 container runtime (VD containerd, dockershim, cri-o...). Ta có thể cấu hình bằng `kubelet --container-runtime --container-runtime-endpoint`

Kata Containers và gVisor là giải pháp cô lập môi trường thực thi (isolation), tạo môi trường riêng biệt để chạy ứng dụng, giảm khả năng ảnh hưởng giữa các container và host.

### Hướng 2: kiểm soát truy cập và hạn chế hành vi (access control & restriction).
AppArmor và seccomp là giải pháp để kiểm soát truy cập và hạn chế hành vi (access control & restriction).​

Kiểm soát truy cập và hạn chế hành vi (AppArmor, seccomp): Giới hạn hành vi của ứng dụng hoặc container trên cùng một hệ thống, không tạo môi trường cô lập nhưng giúp giảm rủi ro tấn công từ bên trong.


### 14.1 Kata container
- Là 1 loại sandbox giúp chạy container trong 1 máy ảo riêng (Firecracker/QEMU) để tăng bảo mật hardware isolation.​
- Mỗi container sẽ có 1 kernel riêng chạy trên 1 lớp phần cứng ảo hóa riêng, tách biệt với kernal của máy host
- Do là tạo thêm 1 lớp ảo hóa nên nếu sử dụng trên cloud thì phải enable tính năng nested virtualization (ảo hóa trong ảo hóa)
- Kata Containers sử dụng các máy ảo nhẹ (lightweight VMs) để chạy container, mỗi container có một kernel riêng biệt. Điều này cung cấp mức độ cô lập phần cứng giữa container và host, giúp ngăn chặn các rủi ro khi container bị tấn công hoặc lỗi. 
<img width="696" height="536" alt="image" src="https://github.com/user-attachments/assets/4312d594-231c-4a3e-8162-b702660ff7ad" />

### 14.2 gVisor
- Sandbox của Google dùng user-space kernel, chặn syscall trực tiếp cho workload untrusted, ưu tiên bảo mật hơn tốc độ.​
- GVisor tạo ra 1 kernel riêng (viết bằng golang) nhận tất cả các syscall từ application, sau đó transform, phân loại hoặc skip trước khi gửi tới kernal thật
- GVisor runs trong userspace tách biệt với linux kernal. OCI container runtime của gVisor là runsc
- gVisor là một sandbox kernel, không phải VM. Nó bắt các system call từ ứng dụng và xử lý chúng trong một môi trường cô lập, không cần phần cứng ảo hóa. gVisor cung cấp mức độ cô lập tốt hơn so với các cơ chế truyền thống như seccomp hay AppArmor, nhưng có thể ảnh hưởng đến hiệu năng do chi phí xử lý system call cao hơn. gVisor phù hợp với các ứng dụng cần cô lập nhưng không yêu cầu mức bảo mật cực cao như VM.
<img width="500" height="307" alt="image" src="https://github.com/user-attachments/assets/1cb99efc-c4d6-4b55-975d-890cce32506d" />

### 14.3 Runtime class
- RuntimeClass là một tài nguyên Kubernetes (thuộc nhóm API node.k8s.io/v1) cho phép chọn cấu hình container runtime cụ thể để chạy các container trong Pod, giúp cân bằng giữa hiệu suất và bảo mật.​
- RuntimeClass được định nghĩa cluster-wide với trường handler trỏ đến CRI runtime handler (như runc, gVisor, Kata), và kubelet trên node sẽ sử dụng nó khi Pod chỉ định runtimeClassName. Nếu không chỉ định, Kubernetes dùng runtime mặc định; nếu handler không tồn tại, Pod sẽ fail.​
- Ứng dụng thực tế: Chạy workload nhạy cảm bảo mật với runtime sandbox như gVisor (handler: runsc) hoặc Kata (handler: kata-runtime).​ Hỗ trợ multi-runtime trong cluster, như runc cho hiệu suất cao và VM-based cho isolation mạnh

-  ví dụ định nghĩa RuntimeClass cho gVisor (handler: runsc) và Kata Containers (handler: kata)
```
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor  # Tên dùng trong Pod spec
handler: runsc  # CRI handler trên node
overhead:
  podFixed:
    cpu: 100m
    memory: 128Mi
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata-runtime
nodeSelector:
  node.kubernetes.io/runtime: kata  # Chỉ schedule lên node hỗ trợ Kata [web:21][web:22]
```

- Áp dụng RuntimeClass vào Pod spec bằng runtimeClassName
```
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  runtimeClassName: gvisor  # Dùng gVisor sandbox
  containers:
  - name: app
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: vm-isolated-app
spec:
  runtimeClassName: kata  # Dùng Kata VM isolation
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"] [web:21][web:24]
```

Lưu ý trên worker node cần phải cài đặt gVisor/Kata và cấu hình containerd sử dụng gVisor/Kata
​

### 14.4. AppArmor
AppArmor là một cơ chế bảo mật dựa trên kernel Linux, cho phép định nghĩa các policy chi tiết để giới hạn quyền truy cập của ứng dụng hoặc container vào hệ thống. AppArmor sử dụng các profile để chỉ định ứng dụng được phép làm gì, ví dụ như truy cập file, network hay các tài nguyên hệ thống. AppArmor hoạt động trên cùng một kernel với host và không tạo môi trường cô lập riêng biệt.
​
AppArmor: Định nghĩa policy để giới hạn quyền truy cập của ứng dụng hoặc container đến tài nguyên hệ thống (file, network, process). AppArmor hoạt động trên cùng kernel với host và không tạo môi trường cô lập riêng biệt, chỉ giới hạn hành vi của ứng dụng.

AppArmor được cài đặt sẵn trên Ubuntu

### 14.5. seccomp
seccomp (secure computing mode) là một cơ chế lọc system call của Linux. Nó cho phép giới hạn các system call mà một tiến trình hoặc container có thể thực hiện, từ đó giảm bề mặt tấn công. seccomp thường được dùng kết hợp với các cơ chế khác như AppArmor hoặc SELinux để tăng cường bảo mật. seccomp hoạt động trực tiếp trên kernel, không tạo môi trường cô lập riêng biệt như VM hay sandbox kernel.

seccomp: Lọc và giới hạn các system call mà một tiến trình hoặc container có thể thực hiện, từ đó giảm khả năng tấn công thông qua các lỗ hổng kernel. seccomp hoạt động ở mức kernel, không tạo môi trường cô lập mà chỉ giới hạn hành vi.

## 15. Security context
- Security Context trong Kubernetes là cơ chế định nghĩa các thiết lập quyền hạn và kiểm soát truy cập cho Pod hoặc Container, giúp thực thi nguyên tắc least privilege (quyền hạn tối thiểu).
- Các thuộc tính Security Context có thể set: 
  - runAsUser/runAsGroup/runAsNonRoot: Chỉ định UID hoặc buộc non-root (true/false).
  - capabilities: add/drop Linux capabilities. Ví dụ: drop ALL để loại bỏ quyền thừa, add NET_ADMIN để cho phép config iptables.
  - seLinuxOptions
  - Chế độ privileged/unprivileged. Mặc định container chạy ở chế độ `Unprivileged`, bị giới hạn capability Linux, không truy cập thẳng vào device của host và bị áp các cơ chế như cgroups, seccomp, AppArmor/SELinux, cho dù bên trong container có thể đang là user root. Còn `Privileged` là mode mà user root trong container được map với user root trên host, lúc này container gần như có toàn bộ quyền như một tiến trình root chạy trực tiếp trên host ( truy cập hầu hết device trong /dev, load module kernel, chỉnh sysctl, mount filesystem, can thiệp network host,...). Privileged chỉ nên dùng cho các workload đặc biệt như container quản lý hệ thống, cần thao tác trực tiếp kernel/device (ví dụ Docker-in-Docker, driver/hardware tooling).
  - allowPrivilegeEscalation: dùng để kiểm soát việc process bên trong container có được phép vượt quyền so với process cha hay không.​ Khi allowPrivilegeEscalation: false, kubelet sẽ bật cờ kernel no_new_privs (kiểm tra bằng lệnh `cat /proc/1/status`), nghĩa là mọi child process trong container không thể có nhiều quyền hơn process đã spawn ra nó (không thể lợi dụng setuid, sudo, file capability… để nhảy lên quyền cao hơn).​ Khi allowPrivilegeEscalation: true (giá trị mặc định), container được phép sử dụng các cơ chế đó nếu trong image/binary có hỗ trợ. Nếu container chạy ở chế độ `privileged: true` hoặc có capability cực mạnh như CAP_SYS_ADMIN, Kubernetes luôn coi allowPrivilegeEscalation là true (không thể ép về false), vì lúc này bản thân container đã có quyền rất cao rồi.​ Do đó best practice là với workload bình thường, chạy ở chế độ unprivileged, drop bớt capabilities, chạy non-root, và đặt allowPrivilegeEscalation: false để giảm khả năng exploit leo thang quyền bên trong container
- Security Context có thể định nghĩa ở hai mức: Pod-level (áp dụng cho tất cả container trong Pod) và Container-level (áp dụng riêng cho từng container, ghi đè Pod-level nếu xung đột).​
- Ví dụ YAML cơ bản:

```
spec:
  volumes:
    - name: vol
      emptyDir: {}
  securityContext:
    runAsUser: 1000 #Chạy với user id 1000 (non-root)
    runAsGroup: #Chạy với main group id 3000
    fsGroup: 2000 #Gán group cho shared volumes
  containers:
    - name: my-pod
      image: busybox
      command:
        - sh
        - -c
        - sleep 1d
      resources: {}
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true #Khi đặt là true, kubelet sẽ kiểm tra user chạy trong container và chỉ cho phép container start nếu UID khác 0. Nếu image được cấu hình để chạy bằng root (UID 0) thì Pod sẽ fail với error “image will run as root”
        privileged: true #đặt container ở mode Privileged. Mặc định là false
```
​
## 16. Service mesh
- Mutual TLS (mTLS) là phiên bản nâng cao của giao thức TLS, yêu cầu cả client và server xác thực lẫn nhau bằng chứng chỉ số trước khi thiết lập kết nối an toàn. TLS chỉ xác thực server (một chiều)
- Mặc định trong cụm k8s, các pod giao tiếp với nhau không có mã hóa.
- Service mesh (như Istio, Linkerd) giúp thực thi mTLS giữa các pod, tăng cường bảo mật. Service mesh thực thi mTLS bằng cách triển khai sidecar proxy (như Envoy trong Istio) bên cạnh mỗi workload, tự động chặn và mã hóa traffic giữa các service mà không cần thay đổi code ứng dụng. Bản chất của các sidecar proxy là sidecar container được deploy trong cùng pod với workload container​
- Cơ chế hoạt động: Sidecar proxy của client lấy chứng chỉ từ control plane (như Citadel trong Istio), sau đó thực hiện TLS handshake hai chiều với sidecar proxy của server, trao đổi và xác thực chứng chỉ trước khi cho phép traffic mã hóa đi qua. Control plane quản lý phân phối, xoay vòng chứng chỉ tự động và cấu hình policy (PERMISSIVE hoặc STRICT) để enforce mTLS toàn mesh hoặc namespace cụ thể.​
- Service mesh dùng các rule iptables trong network namespace của pod để chuyển hướng (redirect) traffic đến/đi từ app sang proxy. Nhờ iptables, app chỉ cần kết nối như bình thường (ví dụ gọi http://service-b), nhưng kernel sẽ tự động bắt và chuyển traffic sang proxy, proxy mới thực sự thiết lập kết nối mTLS với service khác. Do đó mọi traffic in/out của app đều đi qua proxy, app không nói chuyện trực tiếp với bên ngoài. Các rule iptables này được thiết lập bằng một initContainer (chạy các lệnh iptables trong pod (network namespace shared), sau đó exit) chạy trước khi app và proxy start.
- Lưu ý để thiết lập được iptables rule, initContainer phải có Linux capability NET_ADMIN.


## 17. Open policy agent (OPA)
- Là một policy engine mã nguồn mở dùng để hiện thực “policy as code” cho nhiều hệ thống, trong đó Kubernetes là use case rất phổ biến. OPA dùng ngôn ngữ khai báo Rego để mô tả các rule cho phép/từ chối, sau đó đánh giá input dạng JSON (request, config…) và trả về quyết định cho hệ thống gọi nó.​
- Mục đích của OPA là để hợp nhất việc thực thi policy (security, compliance, governance) cho nhiều thành phần: Kubernetes, microservices, API gateway, CI/CD pipeline, Terraform….​ Giúp enforce các rule chi tiết hơn so với cơ chế sẵn có như RBAC, ví dụ:
  - cấm privileged container
  - bắt buộc gắn label cụ thể
  - không cho tạo Service type LoadBalancer ở namespace nào đó
  - không cho deploy workload vào default namespace
  - không cho sử dụng image từ các untrusted registry​
- Trong Kubernetes OPA thường chạy kèm với một thành phần tích hợp, ví dụ OPA Gatekeeper hay kube-mgmt, để hook vào admission controller của API server.​ Luồng hoạt động:
  - Khi người dùng kubectl apply một manifest, API server gửi object đó đi qua chuỗi admission controller (built‑in + dynamic) trước khi ghi vào etcd.
  - OPA (thường thông qua Gatekeeper hoặc kube-mgmt) được triển khai như ValidatingAdmissionWebhook; API server gọi webhook này, gửi JSON của object sang OPA để đánh giá policy.​
  - Khi nhận request, OPA tính toán và trả về “allow/deny” kèm message, admission controller dùng kết quả này để chặn hoặc cho qua request.​

### 17.1 OPA Gatekeeper
- OPA Gatekeeper là một admission controller “bọc” OPA thành dạng Kubernetes‑native, giúp quản trị viên interact dễ dàng với OPA bằng file yaml, quản lý các policy như resource trong k8s cluster.​
- Mặc định sau khi cài OPA Gatekeeper ta sẽ có các CRD trong cluster `configs.config.gatekeeper.sh`, `constraintpodstatuses.status.gatekeeper.sh`, `constrainttemplatepodstatuses.status.gatekeeper.sh`, `constrainttemplates.templates.gatekeeper.sh`. Trong đó:
  - ConstraintTemplate để định nghĩa “kiểu policy” (Rego + schema các tham số), ví dụ “cấm container chạy privileged” ⭢  sau khi ta apply ConstraintTemplate thì OPA Gatekeeper sẽ tạo ra CRD được định nghĩa trong template
  - Constraint: là instance cụ thể của ConstraintTemplate sử dụng CRD tạo ra từ ConstraintTemplate, gắn vào nhóm resource/namespace nào, tham số cụ thể ra sao (list namespace được phép, registry được phép…).
- Tính năng bổ sung: Gatekeeper có cơ chế audit định kỳ, quét toàn cluster để phát hiện resource hiện tại vi phạm constraint (không chỉ request mới), giúp bạn xem độ tuân thủ và dọn dẹp cấu hình cũ.
- Để cài OPA Gatekeeper cần đảm bảo không có plugin nào enabled ngoài NodeRestriction (check ở `/etc/kubernetes/manifest/kube-apiserver.yaml` option `--enabled-admission-plugin`)

ValidatingWebhookConfiguration cho phép tạo admission plugin mà không cần phải register với API server (??)

admission webhook là gì (giống admission controller). OPA gatekeeper tạo custom webhook -> mỗi khi pod được tạo ra đều phải pass qua webhook này

Có 2 loại admission webhook là validate và mutate. OPA chỉ làm việc với validate webhook

Luồng hoạt động là tạo template trước, sau đó tạo constraint từ template đấy

<img width="2183" height="689" alt="image" src="https://github.com/user-attachments/assets/0be0bac1-bc12-423a-9959-13d87da1a4cc" />


```
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8salwaysdeny
spec:
  crd:
    spec:
      names:
        kind: K8sAlwaysDeny
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema: #đây là các thông tin để tạo CRD. Bất kỳ CRD đều cần các thông tin này
          properties:
            message:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: | #rego language là ngôn ngữ để tạo policy
        package k8salwaysdeny

        violation[{"msg": msg}] {
          1 > 0 #có thể khai báo nhiều condition. Nếu tất cả condition đều true thì sẽ xem là violation, tức pod sẽ bị deny
          msg := input.parameters.message
        }
```

-> Sau khi ta apply ConstraintTemplate thì OPA sẽ tạo ra CRD K8sAlwaysDeny (define ở line 9), đồng thời ta có thể get được constrainttemplate tên là k8salwaysdeny

-> Từ đó ta có thể tạo constraint resource với kind là K8sAlwaysDeny

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAlwaysDeny #tạo constraint resource với kind là K8sAlwaysDeny
metadata:
  name: pod-always-deny
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    message: "ACCESS DENIED!"
```
-> sau khi apply thì ta get được resource  K8sAlwaysDeny tên là pod-always-deny

Lưu ý OPA chỉ deny/allow new pod chứ không remove violated pod


Rule để force phải dùng docker.io registry
```
apiversion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8strustedimages
spec:
  crd:
    spec:
      names:
        kind: K8sTrustedImages
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8strustedimages

        violation({"msg": msg}) := {
            input.review.object.spec.containers
        } | {
            img := input.review.object.spec.containers[_].image
            not startswith(img, "docker.io/")
            msg := sprintf("img starts with 'docker.io/' not trusted image '%v'", [img])
        }
```
**Tự làm require label**



- Admission webhook là một web service (HTTP callback) mà kube‑apiserver gọi tới trong giai đoạn admission để kiểm tra hoặc chỉnh sửa request trước khi object được lưu vào etcd.​
- Admission webhook nhận một object AdmissionReview (chứa toàn bộ resource + metadata request), xử lý theo logic của bạn rồi trả về AdmissionReview response cho apiserver, trong đó cho biết request được phép hay bị từ chối. Có hai loại:​

Mutating admission webhook: được gọi trước, có thể trả về JSONPatch để sửa object (ví dụ inject sidecar, set default field).​

Validating admission webhook: được gọi sau, chỉ có quyền allow/deny (ví dụ enforce policy như Gatekeeper, Kyverno).​

Liên hệ với OPA/Gatekeeper
Khi dùng OPA Gatekeeper, bản thân Gatekeeper chạy như một admission webhook loại Validating; mỗi request tạo/sửa resource sẽ bị apiserver gửi sang cho Gatekeeper evaluate policy rồi mới quyết định lưu hay không. Điều này cho phép bạn mở rộng khả năng kiểm soát của Kubernetes mà không cần sửa code của apiserver.


---

(Tìm hiểu thêm về adumission controller)

admission webhook là một dạng thực thi cụ thể của admission controller.

Mối quan hệ
“Admission controller” là khái niệm tổng quát: đó là bước xử lý trong kube‑apiserver, nơi các plugin được gọi để mutate/validate request trước khi ghi vào etcd.

Có hai nhóm plugin: built‑in (viết sẵn trong code apiserver, ví dụ NamespaceLifecycle, ResourceQuota, PodSecurity…) và dynamic mà bạn tự khai báo.

Admission webhook nằm ở đâu

Admission webhook chính là dynamic admission controller được triển khai bên ngoài apiserver dưới dạng HTTP service; apiserver gọi nó như một plugin trong chuỗi admission.

Cụ thể, hai plugin MutatingAdmissionWebhook và ValidatingAdmissionWebhook là built‑in “khung”, còn mỗi webhook (OPA Gatekeeper, Kyverno, custom webhook bạn viết) là một instance chạy ngoài cluster control‑plane, được cấu hình bằng MutatingWebhookConfiguration/ValidatingWebhookConfiguration.

Tóm lại: mọi admission webhook đều là một phần của cơ chế admission controller, nhưng không phải mọi admission controller đều là webhook (vì còn rất nhiều plugin built‑in chạy nội bộ trong apiserver).


## 18. Secure image
- Khi viết Dockerfile có thể xóa các lệnh không cần thiết `rm -rf /bin/*`
- Dùng các tool như Clair/Trivy, đều là các công cụ mã nguồn mở dùng để scan lỗ hổng bảo mật (CVE) cho container image​

### 18.1 Clair
- Là engine phân tích lỗ hổng cho image OCI/Docker theo kiểu “service”, expose API để registry hoặc hệ thống khác gửi thông tin layer/image sang cho nó scan.
- Tập trung vào static analysis, đối chiếu package trong image với nhiều nguồn database CVE (như distro DB, Red Hat DB) và trả về danh sách vulnerability chi tiết, severity, package ảnh hưởng.
​- Thường được tích hợp phía sau các container registry như Harbor, Quay
- Setup hơi phức tạp (cần DB riêng, service riêng) và scan chậm hơn, bù lại report khá chi tiết.
​
### 18.2 Trivy
- Là tool scan “all-in-one” do Aqua Security phát triển, hỗ trợ scan container image, filesystem, Git repo, Kubernetes cluster, cloud resource, SBOM… cho cả lỗ hổng, misconfiguration và secrets.
​- Thiết kế CLI đơn giản (trivy image, trivy fs, trivy k8s), update DB nhanh, scan tốc độ cao nên rất phù hợp nhúng vào CI/CD pipeline để fail build khi có CVE vượt ngưỡng.
​- Ngoài image, Trivy còn có module kiểm tra IaC (Terraform, K8s YAML) và tạo/scan SBOM nên phạm vi rộng hơn hẳn so với Clair truyền thống
- Ví dụ dùng Trivy để scan kube-apiserver image `docker run ghcr.io/aquasecurity/trivy:latest image k8s.gcr.io/kube-apiserver:v1.19.3`

| Tiêu chí              | Clair                                   | Trivy                                               |
|-----------------------|-----------------------------------------|-----------------------------------------------------|
| Kiểu triển khai       | Service + API, cần DB riêng             | CLI/agent, có thể chạy standalone hoặc trong CI/CD  |
| Phạm vi scan          | Chủ yếu container image                 | Image, filesystem, repo, K8s, cloud, IaC, SBOM      |
| Tốc độ                | Thường chậm hơn                         | Nhanh, phù hợp pipeline                             |
| Mức độ chi tiết       | Report lỗ hổng rất chi tiết             | Chi tiết tốt, thêm misconfig & secrets             |
| Độ dễ dùng            | Setup phức tạp hơn                      | Cài và dùng rất đơn giản                            |
| Tích hợp phổ biến     | Harbor, Quay, registry on‑prem          | CI/CD, Kubernetes, nhiều nền tảng khác              |


## 19. Kubesec
- Là một công cụ opensource chuyên phân tích lỗ hổng bảo mật các tệp cấu hình Kubernetes (như YAML manifests cho Pod, Deployment, Service).
- Kubesec quét và đánh giá rủi ro bảo mật dựa trên các security best practices (chạy container với quyền root, sử dụng image không đáng tin cậy, thiếu giới hạn tài nguyên (CPU/memory), hoặc cấu hình network policy yếu,...), giúp phát hiện lỗ hổng trước khi triển khai.
- Kubesec có thể chạy bằng
  - File binary
  - Docker container
  - Kubectl plugin
  - Admission Controller (kubesec-webhook) 
- Ví dụ sử dụng kubesec dạng container để quét file pod.yaml: `docker run -i kubesec/kubesec:latest scan /dev/stdin < pod.yaml` -> trả về score và các advises

## 20. Conftest OPA
- Là một công cụ opensource thuộc dự án Open Policy Agent (OPA), dùng để kiểm tra và xác thực các tệp cấu hình có cấu trúc như YAML Kubernetes, Terraform, Dockerfiles hoặc JSON.
- Conftest sử dụng ngôn ngữ Rego của OPA để viết các policy-as-code, giúp phát hiện vi phạm chính sách bảo mật, tuân thủ hoặc best practices trước khi triển khai. Nó chạy qua CLI với lệnh conftest test file.yaml, báo cáo pass/fail/warn kèm thông báo chi tiết, ví dụ kiểm tra container không chạy root hoặc thiếu resource limits.
​- Cách sử dụng:
  - Cài đặt qua `go install github.com/open-policy-agent/conftest@latest`, tạo thư mục policy với file Rego (như policy.rego), rồi chạy `conftest test -p policy/ deployment.yaml`
  - Hoặc chạy bằng container `docker run --rm -v $(pwd):/project instrumenta/conftest test deploy.yml`​
- Conftest khác Kubesec ở chỗ dựa vào các policy tự define bằng Rego, có thể xác thực đa định dạng (K8s, TF, Docker,...)
- Ví dụ define rule kiểm tra securityContext, label và image trước khi triển khai:
```
package main

deny[msg] {
    input.kind == "Deployment"
    not input.spec.template.spec.securityContext.runAsNonRoot == true
    msg = "Containers must not run as root"
}

deny[msg] {
    input.kind == "Deployment"
    not input.spec.selector.matchLabels.app
    msg = "Containers must provide app label for pod selectors"
}

denylist = [
    "ubuntu"
]

deny[msg] {
    input[i].Cmd == "from"
    val := input[i].Value
    contains(val[i], denylist[_])

    msg = sprintf("unallowed image found %s", [val])
}

```

## 21. Software supply chain secure
- Là toàn bộ chuỗi “cung ứng” để tạo ra và phân phối một sản phẩm phần mềm: từ mã nguồn của bạn, thư viện open‑source, tool build, CI/CD, registry, đến hạ tầng chạy sản phẩm cho khách hàng.
- Mỗi mắt xích trong chuỗi (thư viện bên thứ ba, pipeline, image registry, plugin…) đều có thể bị tấn công và trở thành cửa hậu đưa malware vào sản phẩm của bạn.
​- Các vụ tấn công nổi tiếng như chèn mã độc vào thư viện npm/PyPI hay chiếm quyền build server đều là tấn công vào software supply chain, nên gần đây mới nổi lên các framework như SLSA, NIST SSDF, cùng kỹ thuật ký code, SBOM, scan dependency, ký/verify container image.

### 21.1. Đảm bảo sử dụng đúng imagẻ
- Trong Kubernetes, best practice là “pin” image bằng digest (SHA‑256 hash) thay vì chỉ dùng tag version, vì digest đảm bảo node luôn kéo đúng một binary image cụ thể, ngay cả khi tag bị đổi hoặc bị xoá trên registry.
- Cách thực hiện: trong manifest K8s chỉ định image ở dạng `repo/app@sha256:...`​

## 21.12. ImagePolicyWebhook
- Là một admission controller tích hợp sẵn trong Kubernetes, dùng để kiểm tra và thực thi chính sách về container images trước khi Pod được tạo. Nó gọi một webhook server bên ngoài để validate images, từ chối những image không tuân thủ quy tắc như từ registry không tin cậy hoặc thiếu tag cụ thể.
​- Cách hoạt động:
  - Kubernetes API server nhận request tạo hoặc cập nhật Pod, nếu request pass Authentication/Authorization thì Admission chain chạy → ImagePolicyWebhook được gọi. Controller tạo một ImageReview object từ Pod manifest spec.containers[].image (extract tất cả container images), sau đó ImageReview được serialize thành JSON và POST qua HTTPS đến webhook endpoint (được config trong kubeconfig file của admission config)
  - Webhook server nhận ImageReview, chạy logic kiểm tra (ví dụ: chỉ allow docker.io, hoặc whitelist registry). Nó trả về ImageReviewStatus với allowed: true/false + reason nếu deny. API server dựa vào đó để accept/reject Pod
  - Nếu webhook không reachable, API server có thể fail closed (reject) hoặc fail open tùy cấu hình.
​- Cấu hình cơ bản: Cần enable plugin trong kube-apiserver manifest: `--enable-admission-plugins=ImagePolicyWebhook` và chỉ định `--admission-control-config-file` trỏ đến file AdmissionConfiguration chứa kubeConfigFile cho webhook (lưu ý cần mount các file AdmissionConfiguration, kubeConfigFile, cert, key vào trong container thì mới sử dụng được). Tạo service webhook với TLS (CA bundle), ví dụ cho nginx-only policy.
- ImageReview là native K8s mechanism - ít linh hoạt hơn nhưng lightweight. ImageReview deprecated dần, khuyến nghị dùng Gatekeeper/OPA cho production


Handson:
- Đổi cấu hình của kube-apiserver: thêm `--enable-admission-plugins=ImagePolicyWebhook` và `adminsion-control-config-file=<path/to/admission/file>`
- Tạo file admission_config định nghĩa AdmissionConfiguration (cần phải mount file này vào trong kube-apiserver ở đường dẫn `path/to/admission/file`)
```
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/admission/kubeconf
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false # control behavior của API server. Khi set là false thì nếu external webhook unreachable API server sẽ không tạo pod. Set là true thì nếu external webhook unreachable API server sẽ cho phép tạo pod
```
- Tạo file kubeconf mà admission_config trỏ đến. File này dùng để lưu thông tin kết nối đến external webhook (cần phải mount file này vào trong kube-apiserver)
```
apiVersion: v1
kind: Config
clusters: #clusters refers to the remote service
- cluster:
    certificate-authority: /etc/kubernetes/admission/external-cert.pem #CA dùng để verify remote service (external webhook), cần phải mount file này vào trong kube-apiserver
    server: https://external-service:1234/check-image  # URL của remote service. Phải dùng https
  name: image-checker

contexts:
- name: image-checker-context
  context:
    cluster: image-checker
    user: api-server

current-context: image-checker-context

users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/admission/api-server-client.pem  # Cert để cho webhook admission controller giao tiếp với api server, cần phải mount file này vào trong kube-apiserver
    client-key: /etc/kubernetes/admission/api-server-client-key.pem     # Key match cert client, cần phải mount file này vào trong kube-apiserver
```

​
ImageReview object do ImagePolicyWebhook Admission Controller tạo ra
​
```
apiVersion: imagepolicy.k8s.io/v1alpha1
kind: ImageReview
spec:
  containers:
  - image: myrepo/myimage:v1
  - image: myrepo/myimage@sha256:bebdb8f14cdc2a8db
  annotations:
    mycluster.image-policy.k8s.io/ticket-34: "break-glass"
    namespace: "myspace"
```


​
​
---

## 22. Runtime security - Behavior analytics at host and container level

<img width="1341" height="740" alt="image" src="https://github.com/user-attachments/assets/bbb8a9fd-c142-41ef-b78d-6c761b0c2cbc" />

(hỏi thêm chatgpt về ý nghĩa của ảnh này)

- Kernal tương tác với hardware
- Syscall interface cung cấp giao diện để tương tác với kernal (vd getpid(), reboot(), kill(), mkdir(),fork(), wait(), init(),...)

Lưu ý: Command và syscall là khác nhau về bản chất.

VD Lệnh mkdir và system call mkdir()
- System call mkdir(): Là một hàm hệ thống trong C, được dùng trực tiếp trong chương trình để tạo thư mục. Khi gọi mkdir(), chương trình sẽ gửi yêu cầu đến kernel để tạo thư mục theo đường dẫn và quyền được chỉ định. Nó không có giao diện người dùng, mà được dùng trực tiếp trong code (ví dụ: mkdir("/path/to/dir", 0755)).
​- Lệnh mkdir: Là một tiện ích dòng lệnh để tạo thư mục từ terminal. Lệnh này sẽ gọi đến system call mkdir() ở tầng dưới để thực hiện việc tạo thư mục, đồng thời hỗ trợ thêm các tùy chọn như -p (tạo thư mục cha nếu cần) hoặc -m (đặt quyền). Khi gọi lệnh mkdir với các tùy chọn như -p hoặc -v, thì hệ thống sẽ xử lý các tùy chọn này ở tầng lệnh, chứ không phải syscall mkdir() trực tiếp xử lý. Cụ thể:
  - Tùy chọn -p: Lệnh mkdir sẽ tự động kiểm tra và tạo tất cả các thư mục cha cần thiết trước khi gọi syscall mkdir() cho từng thư mục, từ thư mục gốc đến thư mục cuối cùng. Nếu không có -p, syscall mkdir() chỉ tạo thư mục con nếu thư mục cha đã tồn tại, nếu không sẽ báo lỗi.
​  - Tùy chọn -v: Lệnh mkdir sẽ hiển thị thông báo chi tiết cho mỗi thư mục được tạo thành công, nhưng syscall mkdir() không có chức năng này. Việc hiển thị thông tin này được thực hiện bởi lệnh, không phải syscall.
​


Những ai gọi được syscall: ứng dụng gọi trực tiếp hoặc thông qua library (đỡ phải reinvent the wheel). Lưu ý process trong container cũng có thể gọi được syscall

Để bắt log process gọi syscall ta có thể dùng strace. Strace còn có thể log và hiển thị signal mà process nhận được (log and display signals received by a process ??)

Ví dụ: 

Lệnh `strace ls` để theo dõi và ghi lại tất cả các system call  mà lệnh ls thực hiện khi chạy. Bạn sẽ thấy output dạng:

```
execve("/bin/ls", ["ls"], 0x7ffd...) = 0
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, /* 8 entries */, 2048) = 200
write(1, "file1  file2\n", 12) = 12
```
Mỗi dòng thể hiện system call, đối số và kết quả trả về.
​

Các tùy chọn hữu ích:
- strace -e trace=file ls: Chỉ theo dõi system call liên quan file.
- strace -c ls: Thống kê số lần gọi và thời gian của từng system call.
- strace -o trace.log ls: Lưu output vào file để phân tích sau.
- -v : verbose
- -f : follow
- -cw : đếm và phân loại syscall
- -p : tìm theo PID
- -P : tìm theo path

## 23. Proc directory
Thư mục /proc trong Linux là một filesystem ảo (procfs) cung cấp thông tin thời gian thực về hệ thống, kernel, processes và tài nguyên phần cứng. Nó không lưu trữ dữ liệu trên đĩa mà được kernel tạo ra động khi truy cập, giúp quản trị viên và ứng dụng dễ dàng theo dõi trạng thái hệ thống. Thư mục /proc có thể xem là interface để giao tiếp với kernal Các file trong /proc thường có kích thước 0 byte nhưng chứa dữ liệu văn bản dễ đọc khi dùng lệnh như cat.
​

Các file hệ thống chính
- /proc/cpuinfo: Hiển thị thông tin CPU như model, tốc độ, cores.
- /proc/meminfo: Chi tiết bộ nhớ RAM, swap đang sử dụng.
- /proc/version: Phiên bản kernel, compiler GCC và distro.
- /proc/mounts: Danh sách các mount point hiện tại.
- /proc/modules: Các kernel modules đã load.
​

Thông tin processes
- Các thư mục con dạng /proc/<PID> (PID là ID tiến trình) chứa dữ liệu cụ thể của process đó. Ví dụ:
  - /proc/<PID>/cmdline: Đối số dòng lệnh khởi chạy process.
  - /proc/<PID>/status: Trạng thái process như memory, CPU usage, UID/GID.
  - /proc/<PID>/stat: Thống kê chi tiết về CPU time, faults.

Ví dụ sử dụng /proc để đọc secret lưu trong etcd:
- Tìm PID của etc (container bản chất vẫn là process và có PID). Lưu ý nếu là pod thường thì phải tìm PID ở trên host của pod đấy
- Bonus: dùng lệnh `strace -p <PID>` để xem các syscall mà 1 process gọi. Thêm option -cw để đếm và phân loại
- Truy cập /proc/<PID>. Trong thư mục này sẽ có các thư mục
  - exe: là symbolic links tới file thực thi etcd
  - fd: chứa các symbolic links (symlinks) đại diện cho tất cả các file descriptors (FD) mà process đang mở. Mỗi symlink có tên là số nguyên (như 0, 1, 2, ...) và trỏ đến file thực tế, socket, pipe hoặc tài nguyên I/O khác mà process đang sử dụng. VD FD 0: stdin (chuẩn input, thường từ bàn phím), FD 1: stdout (chuẩn output, thường ra màn hình), FD 2: stderr (chuẩn error output). File mà link tới /var/lib/etcd/member/snap/db sẽ chứa các secret trong etcd
  - environ là file chứa các env (biến môi trường)
 
## 24. Falco
- Là công cụ giám sát bảo mật runtime mã nguồn mở dành cho Linux, container và Kubernetes. Nó phát hiện hành vi bất thường bằng cách theo dõi syscall kernel, file access và network events dựa trên rules tùy chỉnh​
- Rules mặc định phát hiện threat phổ biến: unauthorized shell, outbound connection lạ, privilege escalation. Khi Falco phát hiện rule vi phạm (ví dụ: shell spawn trong container, ghi vào file trong /etc, chạy lệnh liên quan đến package management trong container), nó gửi alert qua stdout, syslog hoặc tích hợp Prometheus/SIEM.
- Cấu hình của Falco nằm trong /etc/falco/falco.yaml
- Log của Falco nằm trong /var/log/syslog
​- Ví dụ rule detect shell in container:
```
- rule: Detect shell in container
  desc: Alert on shell spawn in container
  condition: container and proc.name in (bash, sh)
  output: Shell spawned in container=%container.name (proc=%proc.name)
  priority: WARNING
```
- Các audit rule của Falco nằm trong /etc/falco/k8s_audit_rules.yaml:
  - create sensitive mount pod: là rule để detect pod có volume được mount từ sensitive directory của host (VD /proc)
  - sensitive_vol_mount là macro (giống như condition). Condition này sẽ bằng true khi 1 trong những sensitive path của node được mount vào trong pod
  - falco_sensitive_mount_images là macro để kiểm tra xem image có phải từ các nguồn trust không

- Hands-on: Thay đổi output của Falco (Thay đổi ở /etc/falco/falco_rules.local.yaml để override file rule gốc) thành TIME,USER_NAME,CONTAINER_NAME,CONTAINER_ID
```
- rule: Detect shell in container
  desc: Alert on shell spawn in container
  condition: container and proc.name in (bash, sh)
  output: >
    %evt.time,%user.name,%container.name,%container.id
  priority: WARNING
```

## 24. Đảm bảo container là immutable trong lifecycle
Khái niệm cơ bản
- Mutable container: là container có thể thay đổi nội dung (thêm, xóa, sửa config, env,...) mà không cần tạo ra một container mới.
​- Immutable container là container image được build một lần, deploy nhiều lần, và không bao giờ thay đổi nội dung sau khi chạy. Khi cần cập nhật (patch, config mới) thì cần build image mới, deploy instance mới, rồi xóa instance cũ – đảm bảo mọi container luôn ở trạng thái "known good". Lợi ích: dễ rollback (chỉ switch traffic về phiên bản cũ), tránh configuration drift, và tăng bảo mật vì không thể inject malicious process vào container đang chạy và khi có vấn đề chỉ cần restart lại container để remove các malicious processes

Cách để tạo immutable container:
- Tạo từ container image: xóa bash/shell, set read-only cho filesystem, chỉ chạy container với non-root user
- ghi đè command của process trong container. VD thay vì command là `nginx` thì ta dùng `chmod a-w -R / && nginx` (chỉ cho read-only vào filesystem)
- sử dụng securityContext và PodSecurityPolices để áp dụng read-only filesystem. Nếu muốn write vào 1 số directories cụ thể thì dùng emptyDir. VD
```
spec:
  containers:
  - image: httpd
    name: immutable
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - mountPath: /usr/local/apache2/logs #nếu không mount như thế này thì process sẽ không có quyền ghi vào /usr
      name: cache-volume
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: cache-volume
    emptyDir: {}
```
- Sử dụng startupProbe, VD để xóa bash hoặc make read-only filesystem trước khi container khởi chạy
- Thay đổi logic sang init container. VD init container khởi chạy và ghi dữ liệu vào volume, sau đó mới mount volume vào container chính theo kiểu read-only -> container chỉ có quyền đọc vào volume




​

Readiness vs Liveness:
- Readiness fail thì pod ở trạng thái NotReady và Service ko forward traffic đến. Tuy nhiên container vẫn chạy bình thuồng
- Liveness fail thì container sẽ bị restart
​​
