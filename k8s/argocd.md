# ArgoCD

### 1. Application

Application là đại diện cho một ứng dụng cụ thể mà bạn muốn triển khai lên Kubernetes.
Application chứa các thông tin:
- Nguồn (source): Git repo, Helm chart, Kustomize, v.v. Khi trỏ path của một Application trong ArgoCD tới một folder (thư mục) trong Git repository, ArgoCD sẽ thực hiện các bước sau:
  - Quét và tải tất cả các manifest file (các file định nghĩa tài nguyên Kubernetes dạng .yaml, .yml, .json) trong thư mục đó về.
  - Kiểm tra sự thay đổi: ArgoCD sẽ liên tục theo dõi thư mục này trong repository. Nếu phát hiện thay đổi (ví dụ: cập nhật, thêm, xóa file manifest), ArgoCD sẽ đánh dấu trạng thái của Application là OutOfSync (không đồng bộ) và sẵn sàng cập nhật lại cluster khi bạn đồng bộ (sync) thủ công hoặc ở chế độ tự động.
  - Render manifest: Nếu thư mục chứa các file manifest "thuần" (plain manifests), ArgoCD sẽ parse, validate rồi sử dụng API Kubernetes để áp dụng (apply) các manifest này lên cluster (ở namespace và cluster bạn cấu hình trong Application). Khi trong folder chỉ định có file kustomization.yaml, ArgoCD sẽ hiểu đây là một ứng dụng sử dụng Kustomize, chạy lệnh kustomize build trên thư mục đó (sử dụng phiên bản Kustomize tích hợp sẵn) và apply các manifest tạo ra lên cụm Kubernetes thông qua API.
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

### 3. ApplicationSet 

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
  - Thêm label/annotation: Chèn thêm label hoặc annotation cho tất cả resource. (commonLabels)
  - Patch: Áp dụng các bản vá (patch) để chỉnh sửa một phần nội dung resource, ví dụ sửa image, số lượng replica, env,... (patchesStrategicMerge/patchesJson6902)
  - Thay thế biến (variable substitution): Thay các giá trị động vào manifest.
  - Tạo mới resource động: Sinh ra ConfigMap hoặc Secret từ configMapGenerator hoặc secretGenerator.
  - Thêm hậu tố hoặc tiền tố vào tên resource (nameSuffix/namePrefix)

#### 4.1. Hướng dẫn dùng Kustomize để tạo manifest cho nhiều môi trường

1. Kiến trúc thư mục chuẩn với Kustomize

Để quản lý manifest cho nhiều môi trường (dev, staging, prod), bạn nên tổ chức cấu trúc thư mục như sau:

```
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
```
base/: Chứa các manifest dùng chung cho mọi môi trường.

overlays/: Mỗi thư mục con là một môi trường, chứa các patch hoặc config riêng biệt.

2. Nội dung các file cơ bản
   
```yaml
base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patchesStrategicMerge: #Dùng để ghi đè các giá trị (ví dụ số replicas, image, env...) cho môi trường dev.
  - patch.yaml
nameSuffix: -dev #Thêm hậu tố vào tên resource để phân biệt môi trường.
```

```yaml
overlays/dev/patch.yaml #File Patch này sẽ ghi đè số replicas và image cho môi trường dev.
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

Tương tự cho staging và prod, chỉ cần thay đổi giá trị phù hợp (ví dụ replicas, image tag, biến môi trường...).

3. Sinh manifest cho từng môi trường

```bash
kustomize build overlays/dev > manifest-dev.yaml #build manifest cho môi trường dev
kustomize build overlays/staging > manifest-staging.yaml #build manifest cho môi trường staging
kustomize build overlays/prod > manifest-prod.yaml #build manifest cho môi trường dev
```
Hoặc có thể apply trực tiếp lên cluster bằng lệnh

```bash
kubectl apply -k overlays/dev
```


#### 4.2. File kustomization.yaml cần phải có cả trong base và overlay

1. Vai trò của kustomization.yaml trong base
   
Base là nơi chứa cấu hình gốc, dùng chung cho mọi môi trường (dev, staging, prod...).
File base/kustomization.yaml liệt kê các resource gốc (deployment, service, configmap,...) và có thể định nghĩa thêm các customizations cơ bản.
File này giúp Kustomize biết cần gom những file nào lại thành một bộ cấu hình hoàn chỉnh ban đầu.

2. Vai trò của kustomization.yaml trong overlay
   
Overlay là thư mục chứa các chỉnh sửa, bổ sung hoặc ghi đè dành riêng cho từng môi trường.
File overlays/dev/kustomization.yaml (hoặc tương tự cho từng môi trường) sẽ:
- Tham chiếu đến base (thông qua trường resources hoặc bases).
- Chỉ định các patch, biến môi trường, labels, annotation, v.v. đặc thù cho môi trường đó.
-> Nhờ file này, bạn có thể áp dụng các thay đổi mà không làm ảnh hưởng đến cấu hình gốc.

3. Lý do cần cả hai file
   
Mỗi cấp (base và overlay) là một đơn vị cấu hình độc lập, Kustomize sẽ gom và xử lý từng cấp thông qua file kustomization.yaml tương ứng.
Khi bạn build overlay, Kustomize sẽ:
- Đọc file kustomization.yaml trong overlay để biết cần lấy base nào và áp dụng patch gì.
- Đọc tiếp file kustomization.yaml trong base để biết các resource gốc cần gom lại.
- Kết hợp, biến đổi và xuất ra manifest hoàn chỉnh cho môi trường bạn chọn.

#### 4.3. Trường resources trong file kustomization.yaml của thư mục overlay
   
- Trong file kustomization.yaml của thư mục overlay, trường resources không bắt buộc chỉ tham chiếu đến thư mục base, mà có thể tham chiếu trực tiếp đến từng file cụ thể trong base, miễn là các file đó nằm trong phạm vi truy cập hợp lệ (thường là cùng repo hoặc không bị hạn chế bởi chính sách bảo mật).
Tuy nhiên, cách phổ biến nhất vẫn là tham chiếu đến cả thư mục base. Khi đó, toàn bộ các resource được liệt kê trong base/kustomization.yaml sẽ được overlay kế thừa.

- Khi trường resources trong overlay tham chiếu ít hoặc nhiều resource hơn base
  - Overlay tham chiếu ít resource hơn so với base: Khi overlay chỉ tham chiếu một phần resource của base (ví dụ chỉ lấy deployment.yaml mà không lấy service.yaml), kết quả build cuối cùng chỉ chứa các resource mà overlay đã chỉ định. Các resource khác có trong base nhưng không được overlay liệt kê sẽ không xuất hiện trong manifest đầu ra của overlay.
  - Overlay tham chiếu nhiều resource hơn so với base: Overlay hoàn toàn có thể bổ sung thêm resource mới (ví dụ: thêm file monitoring.yaml hoặc volume.yaml chỉ cho môi trường prod/dev). Khi đó, manifest build ra sẽ là tổng hợp của các resource từ base (nếu có) và các resource mới mà overlay bổ sung.

#### 4.4. Lưu ý khi sử dụng ArgoCD Application

- Luôn trỏ spec.source.path ArgoCD Application vào đúng thư mục overlay của môi trường bạn muốn deploy (ví dụ: overlays/dev, overlays/prod), không để path trỏ vào thư mục mẹ chứa cả base và overlays để tránh lỗi và đảm bảo cấu hình môi trường chính xác theo thiết kế Kustomize

#### 4.5. patchesStrategicMerge, patchesJson6902 và patches

Khi cần chỉnh sửa (patch) resource trong Kustomize overlays, có thể khai báo trong file kustomization bằng 1 trong 3 cách: patchesStrategicMerge, patchesJson6902 hoặc patches. Mỗi cách có ưu điểm và tình huống sử dụng riêng.

1. patchesStrategicMerge
   
Đặc điểm: Dùng các file YAML, chỉ cần khai báo trường muốn thay đổi; phần chưa đề cập vẫn giữ nguyên như trong base. Nên dùng khi muốn đổi hoặc bổ sung các trường đơn giản, kiểu cấu trúc (ví dụ thay replica, sửa image...), nhất là với object hoặc array nhỏ.

Ưu điểm: Dễ viết, không cần hiểu sâu về path JSON, dễ cho team member cùng bảo trì.

Nhược điểm: Không mạnh khi thao tác sâu với array hoặc cần thêm/xóa phần tử cụ thể trong một list (ví dụ, xóa 1 environment variable nhất định trong container).

Ví dụ

kustomization.yaml

```yaml
resources:
  - deployment.yaml
patchesStrategicMerge:
  - patch.yaml
```

patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: my-nginx
          image: nginx:alpine
```

2. patchesJson6902
   
Đặc điểm: Dùng file JSON (hoặc inline), khai báo patch theo chuẩn RFC 6902 (các thao tác add, remove, replace, move, copy, test). Nên dùng khi muốn thay đổi chính xác/truy cập sâu vào cấu trúc resource, đặc biệt là patch vào các mảng (xóa hoặc sửa phần tử cụ thể), hoặc xóa hẳn một trường.

Ưu điểm: Cực kỳ chính xác, thao tác tốt với array và nested field.

Nhược điểm: Cú pháp phức tạp và khó đọc với người mới, phải xác định đúng path.
Ví dụ

kustomization.yaml:
```yaml
resources:
  - deployment.yaml
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx
    path: patch.json
```

patch.json

```yaml
- op: replace #ngoài replace thì còn add, remove, move, copy, test
  path: /spec/replicas
  value: 3
- op: replace #ngoài replace thì còn add, remove, move, copy, test
  path: /spec/template/spec/containers/0/image
  value: nginx:stable
```

3. patches
   
Đặc điểm: Trường tổng quát (hiện đại, nền tảng các phiên bản Kustomize mới) cho phép khai báo patch kiểu YAML (strategic) hoặc JSON6902, cả dạng file ngoài hoặc inline. Nên dùng khi bạn cần tối ưu code base cho team: quản lý patch tập trung, dùng linh hoạt cả hai loại patch trên, tận dụng full sức mạnh của từng tình huống. Ưu tiên dùng trên các bản Kustomize mới

Ưu điểm: Kết hợp cả hai phương pháp trên trong một trường duy nhất, có thể target resource theo nhiều cách nâng cao. Cho phép khai báo patch và target trực tiếp trong kustomization.yaml mà không cần file ngoài nếu muốn (2 cách trên chỉ có thể khai báo qua file ngoài, không thể khai báo inline)

Ví dụ

kustomization.yaml

```yaml
resources:
  - deployment.yaml

patches:
  # Strategic merge patch dùng file YAML
  - path: increase_replicas.yaml
    target:
      kind: Deployment
      name: my-nginx

  # Strategic merge patch inline
  - target:
      kind: Deployment
      name: my-nginx
    patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-nginx
      spec:
        replicas: 3

  # JSON6902 patch dùng file YAML (nội dung JSON)
  - path: patch_memory.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx

  # JSON6902 patch inline
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: httpd:alpine
    target:
      kind: Deployment
      name: my-nginx
```

Patch YAML (increase_replicas.yaml):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 6
```

Patch JSON (patch_memory.yaml)

```yaml
- op: replace
  path: /spec/template/spec/containers/0/resources/limits/memory
  value: 512Mi
```
#### 4.6. Loại bỏ (bỏ bớt) resource trong base khi sử dụng overlay với Kustomize

Cách làm:
- Tạo một file patch với chỉ thị $patch: delete để "xóa" resource không mong muốn khỏi overlay.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <tên service trong base>
$patch: delete
```

- Khai báo patch này trong phần patches (hoặc patchesStrategicMerge/patchesJson6902 tùy phong cách) của kustomization.yaml trong overlay.
```yaml
resources:
  - ../../base
patches:
  - path: delete-service.yaml
    target:
      kind: Service
      name: <tên service trong base>
```

#### 4.7. configMapGenerator và secretGenerator trong Kustomize

1. configMapGenerator trong Kustomize
- Mục đích: Dùng để tạo nhanh các ConfigMap (nguồn cấu hình không nhạy cảm) từ file, chuỗi key-value hoặc biến môi trường (.env) cho các ứng dụng trên Kubernetes mà không cần tự viết YAML thủ công. Khi nội dung của configmap thay đổi, Kustomize sẽ tự động tạo version mới của ConfigMap với tên có hậu tố hash (ví dụ: example-configmap-abcdf1234). Đồng thời manifest của các Pod sử dụng configMap này cũng được thay đổi để sử dụng version mới của ConfigMap -> điều này trigger rolling updates tự động cho các Pods đang sử dụng configMap đó
 
- Cách sử dụng: định nghĩa trong file kustomization.yaml với ba dạng phổ biến:

Từ file:

```yaml
configMapGenerator:
  - name: example-configmap
    files:
      - application.properties
```
Từ biến môi trường (.env):

```yaml
configMapGenerator:
  - name: frontend-configmap
    envs:
      - .env #Nội dung file .env sẽ thành các key-value trong ConfigMap.
```

Từ literal:

```yaml
configMapGenerator:
  - name: example-configmap
    literals:
      - FOO=Bar
      - DEBUG=true
```
- Có thể dùng thuộc tính generatorOptions để thêm nhãn, annotation hoặc tắt việc gắn hash khi cần.


2. secretGenerator trong Kustomize
Mục đích: Dùng để tạo ra Kubernetes Secret (dữ liệu nhạy cảm như mật khẩu, TLS cert…) từ file, biến môi trường hoặc literal, trực tiếp từ Kustomize mà không phải tự encode sang base64 và viết YAML thủ công.

Cách sử dụng: Định nghĩa trong file kustomization.yaml với các dạng nguồn dữ liệu tương tự ConfigMap:

Từ file:
```yaml
secretGenerator:
  - name: my-secret
    files:
      - password.txt
```

Từ biến môi trường (.env):

```yaml
secretGenerator:
  - name: my-secret-env
    envs:
      - .env
```

Từ literal:

```yaml
secretGenerator:
  - name: database-password
    literals:
      - password=pass
```

- Secret cũng được tạo mới với hậu tố hash mỗi khi dữ liệu thay đổi để thuận tiện quản lý và cập nhật Pod tự động nếu cần.
- Nếu tắt tính năng hậu tố hash, cần chủ động cập nhật pod để nhận secret mới.
- Dữ liệu trong Secret luôn được tự động base64 hóa.

### 5. ArgoCD hook

Dùng để thực thi các tác vụ theo các giai đoạn khác nhau trong quá trình triển khai ứng dụng. Có các loại hook:
- PreSync hook: thực thi các tác vụ trước khi ArgoCD triển khai ứng dụng
- Sync hook: Thực thi cùng lúc với việc áp dụng các manifest ứng dụng, sau khi tất cả PreSync hook thành công.
- PostSync hook: Thực hiện sau khi ứng dụng được áp dụng hoàn tất và ở trạng thái khỏe mạnh (Healthy), thích hợp cho các tác vụ kiểm tra, gửi thông báo hay các bước hoàn thiện sau khi triển khai.
- SyncFail hook: Thực thi khi quá trình đồng bộ (sync) thất bại, dùng để xử lý rollback, dọn dẹp tài nguyên hay gửi cảnh báo khi deploy gặp lỗi.
- PostDelete hook: Thực thi sau khi tất cả tài nguyên ứng dụng bị xóa, dùng để làm sạch hoặc dọn dẹp dữ liệu liên quan

Ví dụ dùng PreSync hook:

Cách thực hiện: 
- Tạo một manifest Kubernetes cho Job hoặc Pod chạy script bạn muốn. Nội dung có thể là một Job thực thi file shell script, Python, hoặc bất kỳ container nào có logic bạn mong muốn.
- Thêm annotation `argocd.argoproj.io/hook: PreSync` vào manifest, annotation này cho ArgoCD biết đây là tài nguyên cần chạy trước giai đoạn sync, tức trước khi deploy các thành phần còn lại của ứng dụng.
- Thêm file manifest này vào repo Git mà ArgoCD theo dõi ứng dụng.
- Khi ArgoCD phát hiện thay đổi, nó sẽ tự động thực thi job này trước khi apply các resource còn lại.

```
apiVersion: batch/v1
kind: Job
metadata:
  generateName: custom-script-
  annotations:
    argocd.argoproj.io/hook: PreSync
spec:
  template:
    spec:
      containers:
        - name: script-runner
          image: alpine
          command: ["/bin/sh", "-c"]
          args: ["echo 'Hello from pre-sync!'; ./your-script.sh"]
      restartPolicy: Never
```
