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


### 13.2 Cch giúp ứng dụng của bạn “sống sót” khi nâng cấp cluster Kubernetes hoặc hạ tầng bên dưới.​

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


# 14. Bảo mật container runtime

Về cơ bản container chỉ là các process được cô lập bằng namespaces/cgroups, nhưng bản chất các process này vẫn chạy trên Linux kernel của host và có thể thực hiện system call trực tiếp xuống kernel giống như process bình thường

<img width="1065" height="530" alt="image" src="https://github.com/user-attachments/assets/db8642c8-fa18-4d2f-a9f7-f4a43638576d" />


System call có thể coi như là API mà kernal cung cấp để process có thể giao tiếp với kernal

Để bảo vệ kernal, ta có thể thêm 1 lớp sandbox ở giữa. Sandbox là một môi trường cô lập (hay "hộp cát") được sử dụng để chạy phần mềm, ứng dụng hoặc mã đáng ngờ mà không ảnh hưởng đến hệ thống chính, thường áp dụng trong bảo mật máy tính. Sanbox trong trường hợp này có thể xem là 1 lớp bảo mật để hạn chế tấn công. sandbox container runtime cung cấp lớp cách ly bảo mật cao hơn so với runc truyền thống, ngăn chặn container truy cập trực tiếp kernel host để giảm rủi ro escape
<img width="1135" height="488" alt="image" src="https://github.com/user-attachments/assets/fbf25a53-32fd-4731-bafe-db6b8678628f" />


Kubelet tại 1 thời điểm chỉ có thể chạy 1 container runtime (VD containerd, dockershim,...). Ta có thể cấu hình
