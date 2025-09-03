Giải thích chi tiết sự khác nhau giữa các trường cấu hình etcd
1. --advertise-client-urls
Đây là địa chỉ mà node etcd sẽ thông báo cho các client (ví dụ: kube-apiserver, công cụ quản trị, hoặc các node khác) để kết nối tới node etcd này.

Địa chỉ này sẽ xuất hiện trong thông tin cluster, giúp các thành phần khác biết endpoint nào để truy cập dịch vụ etcd của node này.

Thường là IP private của node, port 2379.

2. --listen-client-urls
Là danh sách các địa chỉ mà etcd sẽ mở (listen) để nhận kết nối từ client.

Thường bao gồm cả 127.0.0.1 (localhost) và IP private thực của máy.

Nếu bạn muốn etcd chỉ nhận kết nối từ localhost, chỉ cần để 127.0.0.1. Nếu muốn các node khác truy cập được, phải thêm IP private.

3. --initial-advertise-peer-urls
Là địa chỉ mà node etcd này sẽ thông báo cho các peer (các thành viên khác trong cụm etcd) để các peer biết cách kết nối peer-to-peer với node này.

Dùng trong quá trình khởi tạo cluster hoặc khi thêm node mới.

Thường là IP private của node, port 2380.

4. --listen-peer-urls
Là danh sách các địa chỉ mà etcd sẽ mở để nhận kết nối từ các peer (các node etcd khác).

Địa chỉ này phải là IP mà các node khác trong cụm etcd có thể truy cập được.

Thường là IP private của node, port 2380.

5. --initial-cluster
Là danh sách tất cả các thành viên ban đầu của cụm etcd, định dạng name=peerURL, phân tách bằng dấu phẩy.

Giúp các node etcd biết được các peer ban đầu để hình thành cluster và đồng bộ dữ liệu.

Ví dụ:
master1=https://192.168.1.10:2380,master2=https://192.168.1.11:2380,master3=https://192.168.1.12:2380



Tóm tắt:

Các trường advertise dùng để thông báo địa chỉ cho các thành phần khác biết cách kết nối đến node etcd này.

Các trường listen xác định etcd sẽ mở các cổng nào để nhận kết nối.

initial-cluster giúp các node etcd nhận diện nhau và hình thành cluster phân tán.

---
Tại sao các trường này thường có cùng một IP? Có bao giờ khác nhau không?
Thông thường, các trường này sẽ dùng cùng một IP là IP private thực của node etcd (ví dụ: 192.168.1.10) để đảm bảo các thành phần khác và các peer trong cluster đều có thể kết nối đến node này.

Các trường listen- là nơi etcd lắng nghe kết nối; các trường advertise- là địa chỉ mà etcd thông báo cho các thành phần khác biết để kết nối tới nó. Thường, chúng nên giống nhau để tránh nhầm lẫn và lỗi mạng.

Có thể khác nhau trong một số trường hợp đặc biệt:

Nếu node etcd có nhiều interface mạng (multi-homed), bạn có thể muốn etcd chỉ lắng nghe trên một interface nhất định (ví dụ: chỉ trên mạng nội bộ).

Nếu bạn muốn etcd chỉ chấp nhận kết nối client từ localhost nhưng peer từ IP private, bạn có thể cấu hình --listen-client-urls=https://127.0.0.1:2379 và --listen-peer-urls=https://192.168.1.10:2380.

Trong môi trường NAT hoặc cloud, đôi khi địa chỉ quảng bá (advertise) phải là IP public hoặc IP của load balancer, còn listen là IP private.

---

Khi một node (máy chủ) có nhiều địa chỉ IP (multi-homed), việc chọn IP nào để điền vào các trường cấu hình như --advertise-client-urls, --listen-client-urls, --initial-advertise-peer-urls, --listen-peer-urls của etcd hoặc các thành phần control plane trong Kubernetes không phải là tự động mà thường dựa vào quyết định của người quản trị hoặc tham số cấu hình khi khởi tạo cluster.

Nguyên tắc chọn IP
Bạn phải chọn thủ công IP phù hợp nhất cho cluster, thường là IP thuộc interface mạng nội bộ (LAN, private network) mà các node khác trong cluster có thể truy cập được.

Không nên dùng IP public, IP NAT, hoặc IP loopback (127.0.0.1) cho các trường này, trừ khi toàn bộ cluster cùng nằm trên một máy hoặc cùng mạng NAT đặc biệt.

Nếu bạn dùng công cụ như kubeadm, khi chạy lệnh khởi tạo, bạn có thể chỉ định IP mong muốn với tham số như --apiserver-advertise-address=<IP> hoặc sửa manifest etcd để điền đúng IP.

---

Khi bạn dùng Load Balancer (LB) cho API server trong cụm Kubernetes HA, các địa chỉ IP trong các trường cấu hình etcd (như --advertise-client-urls, --listen-client-urls, --initial-advertise-peer-urls, --listen-peer-urls, --initial-cluster) vẫn là IP thực của từng node master, chứ không phải là IP của LB.

Giải thích chi tiết
1. Các trường cấu hình etcd dùng IP nào?
Tất cả các trường này đều phải dùng IP thực (IP private) của từng node master để các node etcd có thể giao tiếp trực tiếp với nhau qua các cổng 2379 và 2380.

Không dùng IP của LB cho các trường này.
LB chỉ dùng để load balance traffic đến kube-apiserver (port 6443), không dùng cho giao tiếp giữa các thành viên etcd.

2. Vì sao không dùng IP LB cho etcd?
LB chỉ phân phối traffic đến API server của các master node (kube-apiserver:6443) cho client bên ngoài hoặc worker node.

etcd là một cluster phân tán, các thành viên (master nodes) cần biết chính xác IP của nhau để đồng bộ dữ liệu, bầu leader, v.v. Việc đi qua LB sẽ phá vỡ logic peer-to-peer của etcd và có thể gây lỗi nghiêm trọng cho cluster.

Các trường này phải là IP mà các node master có thể kết nối trực tiếp với nhau, không đi qua LB.


### Blue-green

Chiến lược blue-green deployment trong Kubernetes cho phép triển khai ứng dụng mới với rủi ro tối thiểu và đảm bảo không bị gián đoạn dịch vụ bằng cách chạy song song hai môi trường "blue" (phiên bản cũ) và "green" (phiên bản mới).

Cách hoạt động
Blue-green deployment vận hành bằng cách:

- "Blue" là môi trường production hiện tại tiếp nhận toàn bộ traffic người dùng.

- Khi có bản cập nhật, triển khai "Green" — chạy song song phiên bản mới nhưng chưa nhận traffic bên ngoài.

- Tiến hành test, kiểm tra các tính năng và performace trên môi trường "Green".

- Khi mọi thứ ổn định, chuyển traffic từ "Blue" sang "Green" bằng cách cập nhật Kubernetes Service selector (hoặc sử dụng công cụ như Istio, Argo Rollouts).

- Nếu phát sinh lỗi, có thể chuyển nhanh traffic ngược về môi trường "Blue" để rollback

K8s không native hỗ trợ blue-green Kubernetes (mặc định chỉ có Recreate Deployment và Rolling Update), chỉ có thể thực hiện manual:

- Tạo deployment "blue" (phiên bản cũ) và service trỏ tới pod có label blue.

- Tạo deployment "green" (phiên bản mới) với pod có label green, chưa nhận traffic.

- Test môi trường green.

- Khi sẵn sàng, cập nhật service selector chuyển traffic sang pods green.

- Nếu lỗi, chuyển selector lại về pods blue.


Ngoài thao tác thủ công, có thể dùng các công cụ phổ biến:

- Istio VirtualService giúp chuyển traffic giữa blue và green bằng weight, canary hoặc trực tiếp.

- Argo Rollouts hoặc Flagger giúp tự động hóa quá trình release và rollback.


Các chiến lược nâng cao cần triển khai thủ công hoặc công cụ hỗ trợ
Blue/Green Deployment: Cần chạy song song 2 Deployment, dịch vụ và chuyển selector hoặc thay đổi weight của ingress để điều phối traffic thủ công.

Canary Deployment: Cần tạo nhiều Deployment riêng biệt cho các phiên bản, sử dụng các công cụ như Argo Rollouts, Flagger hoặc service mesh (Istio) để dần dần chuyển traffic.

A/B Testing, Shadow Deployment: Không phải chức năng gốc; dùng service mesh hoặc gateway để điều phối traffic và lọc, sao chép request.

Best-Effort Controlled Rollout: Không trực tiếp có trong K8s, thường phối hợp với các công cụ tự động hóa dựa trên số liệu giám sát.




---

Service Mesh là một lớp hạ tầng (infrastructure layer) chuyên dụng giúp quản lý và kiểm soát giao tiếp giữa các dịch vụ trong hệ thống Microservices. Nó cung cấp các tính năng quan trọng như bảo mật, cân bằng tải, giám sát, quản lý lưu lượng và xử lý lỗi mà không cần thay đổi mã nguồn ứng dụng.

Cách hoạt động của Service Mesh thường dựa vào mô hình "sidecar proxy": mỗi dịch vụ trong hệ thống sẽ có một proxy riêng chạy bên cạnh (sidecar container) để điều phối toàn bộ lưu lượng mạng giữa các dịch vụ. Các proxy này sẽ chịu trách nhiệm mã hóa, xác thực, cân bằng tải, thử lại khi lỗi, ngắt mạch (circuit breaker) và thu thập số liệu giám sát, giúp việc giao tiếp dịch vụ được an toàn, tin cậy và hiệu quả.

Service Mesh tách biệt các phần quản lý giao tiếp (như bảo mật, định tuyến, giám sát) ra khỏi mã nguồn dịch vụ chính, giúp giảm phức tạp cho developer đồng thời tăng cường khả năng quản lý và vận hành hệ thống lớn với nhiều dịch vụ nhỏ (microservices) một cách linh hoạt và tự động.



### Cách chọn node deploy pod trong Kubernetes


| Tiêu chí                  | nodeSelector                                 | nodeAffinity                                       | taints và tolerations                            |
|---------------------------|----------------------------------------------|---------------------------------------------------|--------------------------------------------------|
| Mục đích chính            | Lựa chọn node dựa trên label đơn giản        | Chọn node theo quy tắc linh hoạt dựa trên label   | Đẩy/đuổi pod khỏi node bằng taint + kiểm tra toleration |
| Cách hoạt động            | Pod chỉ định key-value label phải có         | Định nghĩa điều kiện bắt buộc hoặc ưu tiên label  | Node gắn taint, pod cần có toleration tương ứng   |
| Độ linh hoạt              | Thấp (chỉ match đúng label)                  | Cao (nhiều điều kiện + ưu tiên/bắt buộc)          | Cao (kiểm soát scheduling mạnh hơn)               |
| Tác động scheduling       | Scheduler chỉ deploy pod lên node đúng label | Bắt buộc hoặc ưu tiên node phù hợp                | Pod không có toleration sẽ bị đẩy khỏi node có taint |
| Phạm vi kiểm soát         | Chọn node đơn giản dựa trên label            | Phân bổ pod rất linh hoạt                         | Kiểm soát node cho pod đặc biệt, có thể “cấm” pod |
| Dễ sử dụng                | Đơn giản                                     | Phức tạp hơn, nhiều lựa chọn quy tắc              | Cần cấu hình converged cả node và pod             |
| Ví dụ khai báo            | `nodeSelector: {"disktype": "ssd"}`          | `nodeAffinity` với nhiều điều kiện                 | Node taint & toleration trong pod spec            |




### Kiến trúc một cụm Kubernetes (k8s) có tính khả dụng cao (high availability - HA)

Kiến trúc một cụm Kubernetes (k8s) có tính khả dụng cao (high availability - HA) thường bao gồm các thành phần và thiết kế chính sau:

##### Kiến trúc Control Plane với tính khả dụng cao
- Cấu hình multi-master: sử dụng ít nhất 3 node master (control plane nodes) để tránh điểm lỗi đơn. Các thành phần chính như API server, controller manager và scheduler chạy trên các master node này.

- etcd cluster phân tán: etcd là kho lưu trữ trạng thái của toàn bộ cụm, cần được triển khai thành một cụm riêng biệt với từ 3 đến 5 node để đảm bảo tính nhất quán và chịu lỗi tốt.

Hai mô hình phổ biến:

  + Stacked control plane nodes: etcd cùng chạy trên các node master, dễ quản lý nhưng có thể ảnh hưởng hiệu năng.

  + External etcd cluster: etcd chạy trên các node riêng biệt, tăng hiệu năng và độ ổn định nhưng cần hạ tầng phức tạp hơn.

- Sử dụng load balancer như HAProxy để phân phối yêu cầu API server đến các control plane nodes, đảm bảo không có điểm nghẽn hoặc lỗi đơn. HAProxy được cấu hình như một proxy TCP ở cổng 6443 (cổng mặc định của Kubernetes API server), nó định tuyến các yêu cầu đến nhiều API server trên các node master bằng thuật toán cân bằng tải (như round-robin). Nếu hệ thống có nhiều instance HAProxy để chịu tải cũng như tăng dự phòng, sử dụng Keepalived để cung cấp cơ chế Virtual Router Redundancy Protocol (VRRP), tạo một địa chỉ IP ảo (Virtual IP - VIP) được chia sẻ giữa nhiều node chạy HAProxy -> Giúp phòng tránh downtime nếu một node load balancer bị lỗi mà không gây gián đoạn truy cập API server Kubernetes.

- Các master node hoạt động theo mô hình active-active (API server) và active-standby (controller manager, scheduler) với cơ chế bầu lãnh đạo (leader election).

##### Node worker và các chiến lược HA
- Node worker nên được phân bổ đồng đều qua nhiều vùng sẵn sàng (availability zones) để tránh mất dịch vụ khi vùng đó bị lỗi.

- Node pools cho phép quản lý nhóm node đồng nhất, dễ mở rộng, giúp duy trì sức chịu tải và dự phòng.

- Ứng dụng trong cluster cần chạy nhiều bản sao (replica) để luôn có pod thay thế khi có pod bị lỗi.

##### Các yếu tố khác trong kiến trúc HA
- Triển khai cơ chế tự động mở rộng (autoscaling) cho pod và node để ứng phó với biến động tải.
- Persistent storage phải có khả năng chịu lỗi, dùng hệ thống lưu trữ phân tán hoặc lưu trữ đã được cấu hình replica, tránh mất dữ liệu.
  + Lưu trữ phân tán (Distributed Storage): Giúp dữ liệu được nhân bản trên nhiều node lưu trữ, nếu một node lỗi, dữ liệu vẫn còn trên các node khác. Các hệ thống lưu trữ phân tán phổ biến trong Kubernetes: Ceph RBD, GlusterFS, Rook, OpenEBS, Longhorn. Ceph nổi bật với khả năng cung cấp block storage phân tán, có tính sẵn sàng cao và hỗ trợ dynamic provisioning qua CSI (Container Storage Interface). OpenEBS và Longhorn là các giải pháp lưu trữ native cho Kubernetes, dễ triển khai, tích hợp tốt, phù hợp cho các ứng dụng vừa và nhỏ, có khả năng tự động nhân bản, snapshot, backup dữ liệu.
  + Kubernetes sử dụng PersistentVolume (PV) và PersistentVolumeClaim (PVC) để tách biệt cung cấp lưu trữ và yêu cầu sử dụng. Việc cấu hình StorageClass giúp động cơ lưu trữ có thể tự động cấp phát (dynamic provisioning) các volume theo yêu cầu, tăng tính linh hoạt và khả năng mở rộng.




### Số lượng master node trong cụm Kubernetes
Số lượng master node trong cụm Kubernetes thường được khuyến nghị là số lẻ để đảm bảo nguyên tắc quorum trong quá trình bầu chọn leader và duy trì tính khả dụng cao (HA). Bầu chọn leader giúp đảm bảo chỉ có một thực thể điều khiển duy nhất tại một thời điểm, tránh xung đột thao tác.

Nguyên tắc quorum: Kubernetes dùng thuật toán đồng thuận RAFT cho etcd cluster và các thành phần control plane cần bầu lãnh đạo (leader) điều phối cụm. Quorum là số node tối thiểu phải hoạt động và đồng ý để cluster thực hiện các quyết định nhất quán. Với số master node lẻ, quorum phải lớn hơn một nửa, ví dụ: 3 node -> quorum là 2. Nhờ đó, khi một hoặc một vài node master bị lỗi hoặc mất kết nối, cụm vẫn giữ được quorum, tránh hiện tượng split-brain (chia rẽ trạng thái cluster). Nếu số node là chẵn (ví dụ 2 hoặc 4), thì quorum sẽ bằng nửa số node khiến cluster dễ bị mất quorum khi chỉ mất 1 node, làm cluster không thể bầu leader hoặc thực hiện cập nhật trạng thái.

Trong Kubernetes, quá trình bầu chọn leader diễn ra trong các thành phần control plane, cụ thể là:

- Kube-controller-manager: Trong cluster có thể có nhiều instance chạy kube-controller-manager (để HA), nhưng chỉ có một instance giữ vai trò leader để tránh xung đột khi thực hiện các tác vụ điều khiển cluster như tạo pod, cập nhật trạng thái, xử lý node, v.v.

- Kube-scheduler: Tương tự kube-controller-manager, nhiều instance kube-scheduler có thể chạy nhưng chỉ một scheduler được bầu làm leader để chịu trách nhiệm lên lịch (schedule) pod, tránh tình trạng nhiều scheduler cùng chạy gây xung đột.

- etcd cluster: etcd sử dụng thuật toán Raft để bầu chọn một node etcd làm leader trong cụm. Leader etcd chịu trách nhiệm xử lý các yêu cầu ghi và điều phối sự đồng bộ dữ liệu trên các node etcd còn lại. Lưu ý etcd không hoạt động theo kiểu active-standby thuần túy, mà là một cụm (cluster) phân tán dùng thuật toán đồng thuận Raft:
  + Trong cluster etcd sẽ luôn có một leader tại một thời điểm; các node còn lại là follower.
  + Leader xử lý các yêu cầu ghi (write/update) và đồng bộ trạng thái đến các follower.
  + Các follower nhận bản sao trạng thái, kiểm tra sức khỏe leader và sẵn sàng bầu chọn leader mới khi leader hiện tại gặp sự cố.
  + Khi leader gặp lỗi, các follower tự động tổ chức bầu chọn để chọn ra leader mới (leader election).
  + Các node follower không ở trạng thái standby "thụ động" mà vẫn xử lý các yêu cầu đọc (read) và tham gia đồng bộ, hoạt động liên tục.



Mỗi thành phần control plane có cơ chế leader election riêng và có thể có leader khác nhau trên các node master khác nhau trong cùng một thời điểm. Ví dụ trong một cụm 3 node master, có thể xảy ra trường hợp:
- Controller manager tại node 1 làm leader
- Scheduler tại node 2 làm leader
- Etcd node 3 làm leader.

### Tự động restart Pod khi ConfigMap thay đổi
Trong Kubernetes, Pod sẽ không tự động restart khi ConfigMap thay đổi. Đây là hành vi mặc định vì Kubernetes không có cơ chế tự động khởi động lại Pod khi ConfigMap được cập nhật.

Để đạt được việc tự động restart Pod khi ConfigMap thay đổi, bạn có thể dùng một số cách sau:

- Dùng công cụ bên thứ ba như Reloader: Đây là một Kubernetes Controller chuyên giám sát các ConfigMap và Secret được chỉ định, khi phát hiện thay đổi sẽ tự động khởi động lại Pod (ví dụ qua rollout restart). Bạn chỉ cần cài đặt Reloader và thêm annotation vào Deployment để chỉ định ConfigMap cần giám sát.

- Cách thủ công với Kubernetes:

Cập nhật annotation trong metadata template của Deployment mỗi khi ConfigMap thay đổi để kích hoạt rollout restart:

```
spec:
  template:
    metadata:
      annotations:
        configmap-version: "2"  # tăng giá trị đi kèm mỗi lần ConfigMap thay đổi
```
Hoặc dùng lệnh thủ công để rollout restart:
```
kubectl rollout restart deployment <deployment-name>
```
- Một số công cụ quản lý deployment như Helm có thể hỗ trợ tự động tính toán checksum của ConfigMap rồi dùng checksum đó làm annotation kích hoạt restart khi ConfigMap thay đổi.

Cách thực hiện: Trong template deployment.yaml của Helm, thêm phần annotation chứa checksum của ConfigMap vào phần metadata của Pod template.

Khi nội dung ConfigMap thay đổi, giá trị checksum này sẽ khác đi, dẫn đến thay đổi annotation trên Pod template. Kubernetes coi đó là thay đổi nên sẽ rollout lại Deployment, khởi động lại Pod.

Ví dụ cụ thể trong deployment.yaml của Helm:

```
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
Giải thích:

include sẽ lấy template ConfigMap (configmap.yaml),

sha256sum tính toán checksum chuỗi template đó,

checksum này đặt làm annotation, cứ mỗi lần ConfigMap thay đổi thì checksum này thay đổi => Kubernetes sẽ restart Pod.

Một số lưu ý khi áp dụng:

File configmap.yaml trong template Helm phải chứa chính xác content của ConfigMap đang được sử dụng.






Lưu ý: Nếu bạn mount ConfigMap dưới dạng volume, nội dung file sẽ được cập nhật ngay trong container khi Configmap thay đổi, nhưng nếu mount dưới dạng biến môi trường thì không tự cập nhật.


## Ingress


#### Một hệ thống Kubernetes thông thường sử dụng cả LoadBalancer và Ingress Controller để điều phối traffic từ bên ngoài vào các service như sau:

Kiến trúc tổng quan

https://images.viblo.asia/1bf71410-c93e-46d5-b8b8-6ffe5c5cad04.png

External LoadBalancer chịu trách nhiệm nhận toàn bộ traffic từ phía ngoài (Internet hoặc LAN), sau đó chuyển tiếp tới điểm vào cluster (Ingress Controller) thông qua NodePort hoặc trực tiếp qua IP external.

Ingress Controller (ví dụ: NGINX Ingress, HAProxy Ingress) lắng nghe traffic từ LoadBalancer và định tuyến HTTP/HTTPS đến các service nội bộ, dựa trên các rule được định nghĩa trong resource Ingress.

Mọi routing logic như truy cập nhiều domain/subdomain, url path khác nhau được điều phối tại lớp Ingress.

###### Quy trình luồng dữ liệu

Client truy cập vào domain/app: Gửi request HTTP/HTTPS về địa chỉ IP public hoặc private được gán cho LoadBalancer.

LoadBalancer nhận traffic: Forward toàn bộ traffic lên NodePort của Ingress Controller (hoặc IP/port mà Ingress được expose ra).

Ingress Controller xử lý request: Đọc các rule từ resource Ingress, xác định đích đến dựa vào host/path, sau đó chuyển tiếp về service backend thích hợp bên trong cluster.

Service backend nhận request: Service nội bộ nhận request, chuyển về pod ứng dụng xử lý, trả dữ liệu lại theo chiều ngược.

#### Hệ thống sử dụng MetalLB và Ingress Controller

Downside khi dùng Ingress 
- NodePort expose dịch vụ thông qua cổng cố định (thường trong khoảng 30000-32767) trên từng node, khiến người dùng phải biết IP từng node và cổng đó để truy cập. Điều này không tiện và khó quản lý khi có nhiều node hoặc dịch vụ.
- Để giải quyết vấn đề port unwell-known thì cần dùng Loadbalancer đứng trước -> tốn tài nguyên cho LB và phức tạp

-> Dùng MetalLB để xử lý vấn đề trên

https://nvtienanh.info/_next/image?url=%2Fstatic%2Fimages%2Fcai-dat-metallb-va-ingress-nginx-tren-bare-metal-kubernetes-cluster%2FHomeLab-K8s-Full-Architecture-Baremetal.jpg&w=640&q=75

MetalLB cung cấp IP external riêng biệt cho mỗi service LoadBalancer, giúp người dùng truy cập qua một IP cố định, không phụ thuộc IP node, tiện lợi cho DNS và cấu hình mạng.

Cách MetalLB hoạt động: 

- MetalLB cung cấp địa chỉ IP external từ dải IP được cấu hình cho các service loại LoadBalancer trong Kubernetes trên môi trường bare-metal hoặc on-premises.
- Khi một service kiểu LoadBalancer được tạo, MetalLB sẽ nhận diện và cấp phát một IP từ dải IP pool đã định nghĩa.
- MetalLB có thể hoạt động ở chế độ Layer 2 (L2) hoặc BGP (Border Gateway Protocol) để quảng bá địa chỉ IP đó trong mạng nội bộ hoặc ra các router, switch cho phép truy cập từ bên ngoài.
- Trong mô hình thông dụng, MetalLB phối hợp với Ingress Controller (ví dụ như Contour hoặc NGINX) và proxy (như Envoy) để điều phối luồng traffic từ IP external vào các pod ứng dụng.
  
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.50/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```


Lưu ý khi sử dụng metalLB: Nếu trong cụm đã có một service khác tồn tại trước và chiếm lấy IP trong pool của MetalLB, sau đó bạn mới tạo ingress controller (dạng LoadBalancer), thì ingress có thể không nhận được IP external (nếu pool đã hết IP hoặc đã bị các service khác lấy mất). MetalLB sẽ phân phát IP cho service LoadBalancer nào được tạo trước, dựa theo thứ tự tạo resource. Những service tạo sau, khi pool không còn IP khả dụng, sẽ rơi vào trạng thái pending và không được cấp IP

Cách giải quyết:
- gán IPAddressPool cho từng service/namespace cụ thể bằng các trường serviceSelectors, namespaceSelectors trong spec của IPAddressPool — lúc này chỉ service hoặc namespace được chỉ định mới có quyền lấy IP từ pool đó.
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.100.10/32
  serviceSelectors:
    - matchLabels:
        app: ingress-controller
```
- hoặc sử dụng annotation metallb.io/address-pool (nếu cấu hình nhiều IP pools) để chỉ rõ pool mà service ingress sẽ sử dụng, tách biệt với pool dùng cho các service khác.

```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  annotations:
    metallb.io/address-pool: ingress-pool #Service này sẽ chỉ nhận địa chỉ IP từ pool tên là ingress-pool, không bị ảnh hưởng bởi các pool khác trong cluster.
spec:
  type: LoadBalancer
  selector:
    app: ingress-controller
  ports:
    - port: 80
      targetPort: 80
```
