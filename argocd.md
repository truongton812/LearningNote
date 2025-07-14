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
