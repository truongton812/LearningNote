

1. Chart (Helm Chart)
Chart là một gói (package) chứa mọi thứ cần thiết để triển khai một ứng dụng hoặc một tập hợp tài nguyên trên Kubernetes.

---
Cấu trúc 1 thư mục helm chart

```
basechart
├── .helmignore
├── Chart.yaml
├── charts
├── templates
│ ├── NOTES.txt
│ ├── _helpers.tpl
│ ├── deployment.yaml
│ ├── hpa.yaml
│ ├── ingress.yaml
│ ├── service.yaml
│ ├── serviceaccount.yaml
│ └── tests
│ └── test-connection.yaml
└── values.yaml
```
a. helmignore: chứa các pattern cần ignore khi package helm chart (helm package để tạo ra <name>.tgz)

b. chart.yaml: thông tin metadata về chart (tên, phiên bản, phiên bản của application (thường là docker image tag), mô tả, dependencies)
Trường dependencies trong file Chart.yaml dùng để khai báo các chart khác mà chart hiện tại phụ thuộc vào. Đây là cách để bạn xác định “subchart” hoặc “dependency chart” mà ứng dụng của bạn cần, giúp tự động hóa quá trình cài đặt và đảm bảo đầy đủ các thành phần cần thiết khi triển khai ứng dụng trên Kubernetes
Lợi ích và ứng dụng
Tự động hóa: Triển khai đầy đủ các thành phần phức hợp (ví dụ: ứng dụng chính + database, redis, prometheus…) chỉ với một chart cha.

Quản lý version: Dễ dàng kiểm soát phiên bản chart phụ thuộc bằng trường version.

Tái sử dụng: Dùng lại các chart phổ biến từ cộng đồng hoặc nội bộ như một module nhỏ.

VD
```
dependencies:
  - name: mysql
    version: "9.3.4"
    repository: "https://charts.bitnami.com/bitnami"
  - name: redis
    version: "~14.0.0"
    repository: "https://charts.bitnami.com/bitnami"
```
c. values.yaml: các biến cấu hình mặc định, cho phép tùy biến khi cài đặt ứng dụng

d. Thư mục templates/: chứa các file mẫu (template) định nghĩa các resource của Kubernetes (Deployment, Service, Ingress, v.v…), sử dụng ngôn ngữ Go template để sinh ra manifest hoàn chỉnh dựa trên các giá trị cấu hình từ file values.yaml

e.templates/_helpers.tpl: dùng để tạo các reuseable parts (function)
Lưu ý: trong templates các file bắt đầu bằng _ sẽ không được generate ra thành manifest

e. Thư mục charts/: chứa các chart khác mà chart hiện tại cần dùng, để ở dạng <name>.tgz. Là nơi CHỨA các package/chart phụ đã được tải về, ta có thể tải các file .tgz thủ công về sau đó copy vào thư mục này (hữu dụng khi dùng cho môi trường không có Internet hoặc cần cố định phiên bản). Đây cũng là nơi chứa dependency được khai báo trong trường dependencies của file Chart.yaml sẽ được Helm tự động tải về và lưu trong thư mục charts/ khi bạn chạy lệnh helm dependency update

f. Thư mục tests: dùng để viết test để validate charts có hoạt động không (giống hook trong aws)
---

2. Repo (Helm Repository)
Repo trong Helm (hay còn gọi là Helm Repository) là một kho lưu trữ các Helm charts.

Repo đóng vai trò là nơi lưu trữ tập trung, cho phép bạn publish, version, chia sẻ và tải về các ứng dụng đã được đóng gói dưới dạng chart.

Repo thường được triển khai như một server HTTP, chứa file index.yaml với metadata về tất cả các charts có trong repo cũng như các file chart đã đóng gói.

Bạn có thể sử dụng lệnh Helm (helm repo add, helm repo list, helm repo update) để quản lý các repo, tìm kiếm hoặc cài đặt charts một cách dễ dàng.

Repo có thể là repo công khai (ví dụ: Artifact Hub, Bitnami) hoặc repo cá nhân của một tổ chức để triển khai các ứng dụng nội bộ.

Một repo có thể chứa nhiều chart.

Cụ thể, repo (chart repository) là một kho lưu trữ tập trung chứa các gói chart đã được đóng gói (định dạng .tgz) cùng với file index.yaml—tập hợp metadata liệt kê và trỏ tới từng chart có trong repo đó. Mỗi chart sẽ là một gói ứng dụng khác nhau (ví dụ: nginx, mysql, redis...), có thể được phân phối, quản lý version một cách độc lập trong cùng một repo.

Khi bạn tạo một repo Helm (ví dụ trên GitHub hoặc một hosting HTTP), bạn chỉ cần thêm các file đóng gói chart vào repo, cập nhật lại file index.yaml, thì tất cả các chart này sẽ được cộng đồng hoặc hệ thống tìm kiếm và sử dụng thông qua lệnh helm CLI

Ví dụ:
Khi muốn triển khai Nginx lên Kubernetes bằng Helm, bạn có thể thêm repo Bitnami vào Helm, sau đó cài đặt chart Nginx từ repo đó:

bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-nginx bitnami/nginx
Lúc này, Helm sẽ lấy chart Nginx từ repo Bitnami, áp dụng thông tin cấu hình trong values.yaml, sinh ra các manifest K8S, và triển khai Nginx lên cluster

---

1. helm repo list -> xem các repo đã thêm vào helm. Repo là nơi lưu trữ helm chart
2. helm repo add <ten_repo> <url> -> thêm repo vào helm. Trong đó <ten_repo> là tên trên local của helm chart, còn url là nơi lưu trữ chart
3. helm repo remove <ten_repo> -> xóa repo
4. helm repo update -> update list các chart trong repo
5. helm search repo <pattern> -> Tìm trong các repo hiện đang có theo pattern (bất kỳ pattern gì xuất hiện sẽ được list ra). Thêm option --versions để xem thông tin version của các chart
6. helm install <release-name> <chart> [flags] -> triển khai (deploy) một ứng dụng lên Kubernetes. Trong đó:
- Release name là tên bạn gán cho một lần triển khai (release) của Helm chart lên Kubernetes. Nó giúp Helm theo dõi, quản lý và phân biệt các lần triển khai ứng dụng. Release name là duy nhất trong một namespace trên Kubernetes. Nếu bạn không chỉ định release name, Helm có thể tự sinh tên ngẫu nhiên:
- <chart> có thể là tên chart từ repo (VD bitnami/nginx, lưu ý ở đây là tên repo trên máy local), file chart đóng gói (VD ./mychart-1.0.0.tgz), thư mục chart (VD ./mychart/) hoặc URL
- Các flag có thể sử dụng: --namespace : triển khai lên ns nào  (mặc định là default)
                           --create-namespace: tự động tạo ns nếu chưa tồn tại
                           --version: Chỉ định phiên bản của chart cần cài (theo tag hoặc range). Nếu không chỉ định thì sẽ cài bản latest
                           --atomic: xóa release khi quá trình installation thất bại (nếu không chỉ định thì release sẽ ở trạng thái failed)
                           --set: ghi đè các giá trị mặc định đã được định nghĩa sẵn trong file values.yaml. Nếu --set <key>=null thì sẽ xóa key đấy.
7. helm ls -> liệt kê các release đã triển khai bằng helm. Thêm option --namespace để xem ở 1 ns cụ thể
8. helm upgrade <release-name> <chart> --version 1.0.0 --set "image.tag=0.1.0" -> upgrade chart lên version khác, đồng thời set giá trị cho image.tag, lúc này revision sẽ tăng 1 đơn vị. Lưu ý nếu không chỉ định version thì version của chart vẫn giữ nguyên, chỉ có image là thay đổi
Option --set là để thay thế giá trị trong file values.yaml
9. helm history <release-name> -> xem lịch sử upgrade của release
10. helm status <release-name> --show-resources -> xem thông tin tất cả các resources của release
11. helm rollback <release-name> <revision> -> rollback release về version thấp hơn. Lưu ý khi rollback revision vẫn tăng 1 đơn vị. Nếu không chỉ định revision thì sẽ rollback về revision trước đấy
12. helm uninstall <release-name> -> gỡ release ra khỏi cụm k8s
13. helm uninstall <release-name> --kep-history -> gỡ release ra khỏi cụm k8s, tuy nhiên vẫn giữ history của release và ta có thể rollback lại bất cứ lúc nào
14. helm list --uninstalled -> hiển thị các release đã bị gỡ với tùy chọn --kep-history
15. helm get all <release-name> -> in ra tất cả các resources của chart đã cài đặt kèm file value
16. helm get manifest <release-name> --revision 1 -> chỉ lấy ra manifest của resources của revision 1
17. helm get values <release-name> --revision 1 -> chỉ lấy ra file value truyền vào khi tạo release với revision là 1. Nếu không truyền value thì sẽ trả về null
18. helm create <chart-name> -> tạo ra 1 chart mới với structure chuẩn (chart.yaml, charts, templates, values.yaml)
19. helm lint -> kiểm tra xem chart viết có lỗi không
20. helm template --release-name <release-name> <chart_path> -f values.yaml -n default -> generate ra manifest của chart (chỉ gen ra chứ không deploy)
21. helm dependency list -> xem các dependencies khai báo trong file chart.yaml (lưu ý cần đứng trong root directory của chart để chạy lệnh này)
22. helm dependency build -> build các dependencies khai báo trong file chart.yaml và lưu vào trong thư mục charts dưới dạng <name-version>.tgz
23. helm dependency update -> khi update các dependencies khai báo trong file chart.yaml thì ta chạy lệnh này sẽ sinh ra file <name-version>.tgz mới trong thư mục charts
Chatgpt: Lệnh này sẽ tự động tải về các chart phụ thuộc và lưu vào thư mục charts/ của chart chính, đồng thời tạo file Chart.lock chứa thông tin chính xác về phiên bản thực tế của từng chart phụ thuộc
24. helm package <path_to_chart> -d <directory> -> đóng gói 1 chart và lưu vào <directory> để up lên repo (nếu không chỉ định directory thì lưu vào thư mục hiện tại)
25. helm repo index <path_to_chart> -> tạo file index.yaml. Cần có file này mới push được lên repo
26. helm repo add <chart-name> <repo-url>
27. Option --dry-run: dùng để lấy manifest cuối cùng sẽ được triển khai chứ không chạy thật. VD áp dụng cho helm install, helm upgrade, helm template, helm uninstall
28. Option --debug: enable verbose output (dùng được với đa số lệnh helm)

---


Values Hierarchy
Value trong helm sẽ có thứ tự như sau:
Sub chart values.yaml can be overriden by parents chart values.yaml
Parent charts values.yaml can be overriden by user-supplied value file (-f myvalues.yaml)
User-supplied value file (-f myvalues.yaml) can be overriden by --set parameters
---
Comment trong helm: {{/* comment */}}

Helm builtin object


Root object là dot (.) 
Dưới root object có 6 builtin object:
- .Release: thông tin của release. Các biến hay dùng: .Release.Name, .Release.Namespace, .Release.Revision
- .Value: tất cả value trong file value.yaml và 2 option -f & --set
- .Chart: thông tin từ file chart.yaml
- .Capabilities: thông tin về cụm k8s (phiên bản k8s, phiên bản helm,...)
- .Template: chứa thông tin của current template đang được triển khai (VD tên template, đường dẫn đến template.)
- .Files: cách để truy cập đến các non-template file trong chart
Khi gọi root object bằng syntax {{ . }} thì sẽ in 1 map của 6 object con
Lưu ý quan trọng: range(.) và with(.) thì dấu . sẽ là current context

Dùng function | toYaml để parse dễ đọc hơn

---
Các function của .Files
- Get: function Get của .Files được dùng để lấy nội dung của một file bất kỳ trong chart (trừ các file trong thư mục templates/). Cú pháp {{ .Files.Get "relative/path/to/file }}. VD: {{ .Files.Get "files/config.init" }} -> đọc nội dung file files/config.init. Ứng dụng: lấy nội dung của file đưa vào ConfigMap
- .Glob(path/to/file) đọc nhiều file. Nếu chỉ dùng Glob thì sẽ ra kiểu data dành cho computer, phải chuyển thành human data bằng Config hoặc Secret
  File Glob as Config: {{ (.Files.Glob "config-files/*").AsConfig }} -> hiển thị dưới dạng yaml
  File Glob as Secret: {{ (.Files.Glob "config-files/*").AsSecrets }} -> hiển thị dưới dạng base64
- .Line: ghép tất cả nội dung của file thành 1 dòng, mỗi dòng giờ sẽ thành 1 phần tử của array
  
You can loop through Lines using a range function:
```
data:
  some-file.txt: {{ range .Files.Lines "foo/bar.txt" }}
    {{ . }}{{ end }}
```
---
Cách code
Helm dùng cú pháp của Go.
Ký hiệu "{{ }}" gọi là template Action
Anything in between Template Action {{ .Chart.Name }} is called Action Element. Action Element chỉ có thể là function hoặc builtin object của helm
Anything in between Template Action {{ .Chart.Name }} will be rendered by helm template engine and replace necessary values
Anything outside of the template action will be printed as it is.

- Nối chuỗi: {{ a_var }}-{{ b_var }} -> kết quả a_var-b_var
---
Các function thông dụng
- Pipe: dùng để thực hiện 1 chuỗi các hành động, output của hành động trước sẽ là input của hành động sau. VD {{ .Release.Service | quote | upper | squote }} 

- quote và squote function: dùng để thêm dấu " hoặc '. Có thể đặt trước hoặc dùng qua pipe. VD {{ quote .Release.Service }} hoặc {{ .Release.Service | quote }}
- default: dùng để khai giá trị mặc định, nếu không ghi đè từ file values hoặc option -f hoặc --set thì helm sẽ dùng giá trị mặc định này. Lưu ý nếu string là "", null, hoặc không khai báo thì sẽ bị default ghi đè, nếu numeric là 0, null hoặc không khai báo thì sẽ bị default ghi đè. Tương tự Lists: [], Dicts: {}, Boolean: false
  Syntax: default "foo" .Bar -> coi như function default sẽ cần 2 arguments 
- {{- .Chart.name }}: If a hyphen is added before the statement, {{- .Chart.name }} then the leading whitespace will be ignored during the rendering (xóa tất cả khoảng trắng đằng trước)
  {{ .Chart.name -}}: If a hyphen is added after the statement, {{ .Chart.name -}} then the trailing whitespace will be ignored during the rendering (xóa tất cả khoảng trắng đằng sau)
Lưu ý: nếu ta dùng khoảng trắng để bắt đầu và ngay sau đó đặt template Action thì kết quả generate ra sẽ không có khoảng trắng nào. Nhưng chỉ cần đặt ký tự bất kỳ để bắt đầu, sau đó là khoảng trắng và sau đó là template Action thì kết quả genrate ra sẽ có khoảng trắng
- indent và nindent: indent dùng để thêm khoảng trắng, nindent dùng để xuống dòng và thêm khoảng trắng. VD {{ .Chart.Name | indent 4 }}
- toYaml: là template function dùng để convert list, slice, array,dict hoặc object thành yaml. VD  {{- toYaml .Values.resources | nindent 10}}. Nếu không có toYaml thì kết quả sẽ là 1 map[limits:map[cpu:500m memory:128Mi] requests:map[cpu:250m memory:64Mi]]

- - include (bổ sung sau)
- 
- if else trong helm
{{- if eq .Values.myapp.env "prod" }}
  replicas: 4 
{{- else if eq .Values.myapp.env "qa" }}  
  replicas: 2
{{- else }}  
  replicas: 1
{{- end }}

Dấu "-" để xóa khoảng cách dòng 

- and: trả về true chỉ khi cả 2 cùng true. Syntax: and .Arg1 .Arg2
  {{- if and .Values.myapp.retail.enableFeature (eq .Values.myapp.env "prod") }}

Tham khảo thêm các logic function khác (Helm includes numerous logic and control flow functions including and, coalesce, default, empty, eq, fail, ge, gt, le, lt, ne, not, or, and required.) tại https://helm.sh/docs/chart_template_guide/function_list/ 
