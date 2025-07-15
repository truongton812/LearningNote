# ArgoCD

### 1. Application

Application là đại diện cho một ứng dụng cụ thể mà bạn muốn triển khai lên Kubernetes.
Application chứa các thông tin:
- Nguồn (source): Git repo, Helm chart, Kustomize, v.v.
- Đích (destination): Cụm Kubernetes và namespace sẽ triển khai.
- Project mà Application này thuộc về.

Việc triển khai 1 Application đồng nghĩa với việc triển khai các resource từ git path lên cụm k8s

Mỗi Application chỉ thuộc về một Project duy nhất. Nếu không chỉ định, nó sẽ nằm trong Project default.

Ví dụ YAML

```yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd #Đây là namespace mặc định mà ArgoCD được cài đặt và quản lý các resource Application. Nếu bạn muốn tạo resource Application ở namespace khác, bạn cần cấu hình lại ArgoCD để cho phép quản lý Application ở namespace đó
spec:
  project: my-project   # Chỉ định Project
  source:
    repoURL: 'https://github.com/example/repo'
    path: manifests
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
```

### 2. Project (AppProject)

Project là đơn vị tổ chức cao nhất, dùng để nhóm các Application lại với nhau.
Mỗi Project định nghĩa các giới hạn và quyền kiểm soát cho các Application thuộc về nó, bao gồm:
- Các repository Git được phép sử dụng -> Project có thể chỉ định danh sách các Git repository mà các Application thuộc project này được phép lấy manifest (YAML, Helm chart, Kustomize, v.v.) để deploy. Nếu một Application trỏ đến repo ngoài danh sách cho phép, ArgoCD sẽ từ chối deploy. Cấu hình bằng trường sourceRepos, có thể dùng pattern hoặc wildcard (* và !) để cho phép tất cả, hoặc liệt kê cụ thể từng repo.
- Các cụm (cluster) và namespace mà Application được phép deploy vào. Giúp giới hạn phạm vi ảnh hưởng của Application, tránh việc deploy nhầm sang cluster hoặc namespace không mong muốn. Cấu hình bằng trường destinations trong AppProject, gồm các cặp server (địa chỉ API của cluster) và namespace. Có thể dùng * để cho phép tất cả, hoặc chỉ định cụ thể.
- Loại resource mà Application được phép tạo hoặc chỉnh sửa, ví dụ: chỉ cho phép deploy Deployment, Service, không cho phép tạo Pod trực tiếp hoặc ClusterRole. Tác dụng là đảm bảo Application không thể tạo ra các resource có thể gây rủi ro bảo mật hoặc vượt quyền kiểm soát. Cấu hình bằng trường namespaceResourceWhitelist và clusterResourceWhitelist trong AppProject, liệt kê các nhóm API (apiGroups) và loại resource (kinds) được phép.
- RBAC (Role-Based Access Control) cho từng project, giúp phân quyền chi tiết cho từng nhóm hoặc người dùng. Ý nghĩa: Project cho phép thiết lập quyền truy cập chi tiết cho từng nhóm người dùng hoặc service account đối với các Application trong project đó. Tác dụng: Phân quyền rõ ràng, ví dụ: nhóm A chỉ được xem, nhóm B được tạo/sửa Application, nhóm C được sync/deploy, v.v. Cấu hình: Thiết lập qua ArgoCD RBAC policy, có thể gán quyền theo project, theo user hoặc nhóm (group).

Mặc định, ArgoCD sẽ tạo một Project tên là default. Nếu không chỉ định, Application sẽ thuộc về Project này.

Ví dụ YAML

```yaml
Project:
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
spec:
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: '*'
```

### 3.ApplicationSet 

- Là một Custom Resource (CRD) cho phép bạn tự động sinh ra nhiều đối tượng Application từ một định nghĩa duy nhất, giúp quản lý và triển khai hàng loạt ứng dụng hoặc cùng một ứng dụng lên nhiều cluster, namespace một cách tự động và đồng nhất.
- Đặc điểm nổi bật của ApplicationSet:
  - Tự động sinh Application: ApplicationSet controller sẽ tự động tạo ra các resource Application dựa trên template và các tham số đầu vào (parameters) do bạn định nghĩa trong ApplicationSet.
  - Quản lý đa cụm (multi-cluster) và đa môi trường: Bạn có thể dễ dàng triển khai cùng một ứng dụng lên nhiều cluster hoặc nhiều namespace khác nhau chỉ với một ApplicationSet.
  - Hỗ trợ nhiều nguồn sinh (generator):
      - List generator: Định nghĩa danh sách cụm/namespace cố định.
      - Cluster generator: Tự động lấy danh sách cluster đã đăng ký trong ArgoCD.
      - Git generator: Sinh Application dựa trên cấu trúc thư mục hoặc file trong Git repository.
      - Matrix generator: Kết hợp nhiều generator để sinh ra tập hợp ứng dụng phức tạp hơn.
  - Quản lý tập trung: Thay vì quản lý từng Application riêng lẻ, bạn chỉ cần cập nhật ApplicationSet, mọi Application con sẽ được cập nhật theo.
  - Phù hợp cho monorepo và multi-tenant: Dễ dàng áp dụng với các repo chứa nhiều ứng dụng hoặc môi trường chia sẻ nhiều team.

Ví dụ minh họa
Khai báo một ApplicationSet để deploy ứng dụng "guestbook" lên ba cluster khác nhau:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
    - list:
        elements:
          - cluster: engineering-dev
            url: https://1.2.3.4
          - cluster: engineering-prod
            url: https://2.4.6.8
          - cluster: finance-preprod
            url: https://9.8.7.6
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
    spec:
      project: my-project
      source:
        repoURL: https://github.com/infra-team/cluster-deployments.git
        targetRevision: HEAD
        path: guestbook/{{.cluster}}
      destination:
        server: '{{.url}}'
        namespace: guestbook
```

Kết quả là ArgoCD sẽ tự động tạo ra 3 Application, mỗi Application tương ứng với một cluster được định nghĩa trong danh sách

### 4. Kustomization


Kustomize là một công cụ giúp quản lý và tùy biến các file cấu hình YAML của Kubernetes một cách linh hoạt, đặc biệt khi triển khai ứng dụng trên nhiều môi trường khác nhau như dev, staging, production.

Lưu ý Kustomize không phải là một Custom Resource (CRD) của Kubernetes. Kustomize không tạo ra resource mới trong cluster như các loại CRD (Custom Resource Definition).

Cách hoạt động của Kustomize:
- Bạn tổ chức các file YAML (Deployment, Service, ConfigMap, ...) theo thư mục, rồi tạo một file đặc biệt tên là kustomization.yaml để định nghĩa cách kết hợp, ghi đè, thêm label, đổi namespace, patch, v.v..
- Khi chạy lệnh kubectl apply -k <thư mục>, Kustomize sẽ tìm và đọc file kustomization.yaml trong thư mục đó, xử lý các file YAML (được liệt kê trong mục resources của kustomization.yaml) theo các chỉ định do người dùng khai báo, và trả về bộ manifest đã được tùy biến. Nếu có file YAML trong thư mục nhưng không được liệt kê trong kustomization.yaml, file đó sẽ không được áp dụng khi bạn chạy lệnh kubectl apply -k ....
- Các thao tác phổ biến mà kustomize có thể xử lý:
  - Thêm namespace: Tự động gán namespace nếu resource chưa có.
  - Thêm label/annotation: Chèn thêm label hoặc annotation cho tất cả resource.
  - Patch: Áp dụng các bản vá (patch) để chỉnh sửa một phần nội dung resource, ví dụ sửa image, số lượng replica, env,...
  - Thay thế biến (variable substitution): Thay các giá trị động vào manifest.
  - Tạo mới resource động: Sinh ra ConfigMap hoặc Secret từ configMapGenerator hoặc secretGenerator.

##### Hướng dẫn dùng Kustomize để tạo manifest cho nhiều môi trường

1. Kiến trúc thư mục chuẩn với Kustomize
2. 
Để quản lý manifest cho nhiều môi trường (dev, staging, prod), bạn nên tổ chức cấu trúc thư mục như sau:

my-app/ 
├── base/ 
│   ├── deployment.yaml 
│   ├── service.yaml 
│   └── kustomization.yaml 
└── overlays/  
    ├── dev/ 
    │   ├── kustomization.yaml 
    │   └── patch.yaml 
    ├── staging/ 
    │   ├── kustomization.yaml 
    │   └── patch.yaml 
    └── prod/ 
        ├── kustomization.yaml 
        └── patch.yaml 
        
base/: Chứa các manifest dùng chung cho mọi môi trường.

overlays/: Mỗi thư mục con là một môi trường, chứa các patch hoặc config riêng biệt.

2. Nội dung các file cơ bản
   
a. base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

b. overlays/dev/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base

patchesStrategicMerge: #Dùng để ghi đè các giá trị (ví dụ số replicas, image, env...) cho môi trường dev.
  - patch.yaml
nameSuffix: -dev #Thêm hậu tố vào tên resource để phân biệt môi trường.
```

c. overlays/dev/patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: my-app
          image: my-image:dev
```
Patch này sẽ ghi đè số replicas và image cho môi trường dev.

Tương tự cho staging và prod, chỉ cần thay đổi giá trị phù hợp (ví dụ replicas, image tag, biến môi trường...).

3. Sinh manifest cho từng môi trường
Để build manifest cho môi trường dev:
```bash
kustomize build overlays/dev > manifest-dev.yaml
```
Cho staging:

```bash
kustomize build overlays/staging > manifest-staging.yaml
```
Cho prod:

```bash
kustomize build overlays/prod > manifest-prod.yaml
```

Có thể apply trực tiếp lên cluster:

```bash
kubectl apply -k overlays/dev
```
4. Một số tính năng hữu ích khác
commonLabels: Thêm label chung cho toàn bộ resource của môi trường.

configMapGenerator/secretGenerator: Sinh ConfigMap/Secret riêng cho từng môi trường.

patchesStrategicMerge/patchesJson6902: Hỗ trợ patch mạnh mẽ theo nhu cầu.

5. Bảng tổng kết
Thư mục/file	Vai trò
base/	Chứa manifest gốc dùng chung
overlays/dev/	Chứa patch và config riêng cho dev
overlays/staging/	Chứa patch và config riêng cho staging
overlays/prod/	Chứa patch và config riêng cho prod
kustomization.yaml	File cấu hình chính của từng layer
patch.yaml	File patch ghi đè các thông số cần khác biệt
6. Lưu ý
Không cần tạo lại toàn bộ manifest cho từng môi trường, chỉ cần patch phần khác biệt.

Khi thay đổi ở base, mọi môi trường sẽ nhận được thay đổi chung, giảm lặp code và rủi

---

Vì sao cả base và overlay đều cần file kustomization.yaml?
1. Vai trò của kustomization.yaml trong base
Base là nơi chứa cấu hình gốc, dùng chung cho mọi môi trường (dev, staging, prod...).

File base/kustomization.yaml liệt kê các resource gốc (deployment, service, configmap,...) và có thể định nghĩa thêm các customizations cơ bản.

File này giúp Kustomize biết cần gom những file nào lại thành một bộ cấu hình hoàn chỉnh ban đầu.

2. Vai trò của kustomization.yaml trong overlay
Overlay là thư mục chứa các chỉnh sửa, bổ sung hoặc ghi đè dành riêng cho từng môi trường.

File overlays/dev/kustomization.yaml (hoặc tương tự cho từng môi trường) sẽ:

Tham chiếu đến base (thông qua trường resources hoặc bases).

Chỉ định các patch, biến môi trường, labels, annotation, v.v. đặc thù cho môi trường đó.

Nhờ file này, bạn có thể áp dụng các thay đổi mà không làm ảnh hưởng đến cấu hình gốc.

3. Lý do cần cả hai file
Mỗi cấp (base và overlay) là một đơn vị cấu hình độc lập, Kustomize sẽ gom và xử lý từng cấp thông qua file kustomization.yaml tương ứng.

Khi bạn build overlay, Kustomize sẽ:

Đọc file kustomization.yaml trong overlay để biết cần lấy base nào và áp dụng patch gì.

Đọc tiếp file kustomization.yaml trong base để biết các resource gốc cần gom lại.

Kết hợp, biến đổi và xuất ra manifest hoàn chỉnh cho môi trường bạn chọn.

---

Trong file kustomization.yaml của thư mục overlay, trường resources không bắt buộc chỉ tham chiếu đến thư mục base, mà có thể tham chiếu trực tiếp đến từng file cụ thể trong base, miễn là các file đó nằm trong phạm vi truy cập hợp lệ (thường là cùng repo hoặc không bị hạn chế bởi chính sách bảo mật).
Tuy nhiên, cách phổ biến nhất vẫn là tham chiếu đến cả thư mục base. Khi đó, toàn bộ các resource được liệt kê trong base/kustomization.yaml sẽ được overlay kế thừa.

Cách tham chiếu	Được phép	Ưu điểm	Nhược điểm
Thư mục base	Có	Đơn giản, kế thừa toàn bộ	Có thể dư thừa resource
Từng file cụ thể	Có	Linh hoạt, chọn lọc	Dễ thiếu sót nếu quên file

---

khi trường resources trong overlay tham chiếu ít hoặc nhiều resource hơn base
1. Overlay tham chiếu ít resource hơn so với base
Khi overlay chỉ tham chiếu một phần resource của base (ví dụ chỉ lấy deployment.yaml mà không lấy service.yaml), kết quả build cuối cùng chỉ chứa các resource mà overlay đã chỉ định.

Các resource khác có trong base nhưng không được overlay liệt kê sẽ không xuất hiện trong manifest đầu ra của overlay.

Điều này giúp bạn kiểm soát chính xác resource nào sẽ được deploy cho từng môi trường hoặc mục đích sử dụng cụ thể.

2. Overlay tham chiếu nhiều resource hơn so với base
Overlay hoàn toàn có thể bổ sung thêm resource mới (ví dụ: thêm file monitoring.yaml hoặc volume.yaml chỉ cho môi trường prod/dev).

Khi đó, manifest build ra sẽ là tổng hợp của các resource từ base (nếu có) và các resource mới mà overlay bổ sung.

Điều này rất hữu ích khi cần thêm các thành phần chỉ có ở một số môi trường nhất định mà không làm thay đổi base chung


