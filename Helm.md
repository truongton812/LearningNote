

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
7. helm ls -> liệt kê các release đã triển khai bằng helm (kèm revision, appversion, chart version). Thêm option --namespace để xem ở 1 ns cụ thể
8. helm show/inspect chart <chart_directory> hoặc <chart.tgz> -> xem thông tin chart (appversion, chart version, name,...) trong thư mục chart hoặc chart được đóng gói thành file tar, lệnh này sẽ xem được thông tin ngay cả khi chart chưa được install
8. helm show/inspect value <chart.tgz> -> xem thông tin file value.yaml của file chart.tgz, hữu ích khi không cần giải nén hoặc install file lên thành release
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
24. helm package <path_to_chart> -d <directory> -> đóng gói 1 chart và lưu vào <directory> để up lên repo (nếu không chỉ định directory thì lưu vào thư mục hiện tại). Nếu sau này ta có update chart và muốn lưu thành version mới thì ta sửa thông tin chart version trong chart.yaml rồi chạy lại lệnh helm package, hoặc nếu không sửa chart.yaml thì có thể chỉ định option --version và --app-version khi chạy lệnh helm package (tuy nhiên không nên vì sẽ khó quản lý version)
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

- or: trả về true khi 1 trong 2 bằng true. Syntax: or. Arg1 .Arg2
- not: đảo ngược boolean của argument. Syntax: not .Arg
  VD: {{ if not (eq .Values.myapp.env "prod") }}
- with (là flow control): chỉ định context, thường dùng để chỉ định 1 context mà nhiều action dùng
  VD
```
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }} #lúc này dấu chấm (.) sẽ là context thay thế cho .Values.podAnnotations       
      {{- end }}
```
context có thể dùng ở bất kỳ đâu trong template, VD trong if else, range,...
Khi dùng with để lấy root object thì gọi bằng $.
Lưu ý có 2 cách để gọi root object khi dùng trong with, range
- C1 là dùng $.
- C2 là đặt variable ngoài scope của with hoặc range

#### helm variable
thường dùng với with, range action và named template
cách định nghĩa variable: {{ $<ten_var> := <gia_tri> }}
VD: {{ $chartname := .Chart.Name }}
cách gọi: {{ $<ten_var> }}
lưu ý quan trọng: nếu define variable trong with thì variable đấy sẽ lấy context của with, còn ngoài with thì là lấy Root làm context

#### range
- range trong helm: là for, dùng để lặp 1 list. Khi khai báo range phải khai báo cả context, không thể dùng Root
- Example 1
```
{{- range .Values.namespaces }} 
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .name }}
---  
{{- end }}
```
values.yaml
```
namespaces:
  - name: myapp1
  - name: myapp2
  - name: myapp3 #lưu ý các key phải là "name", nếu dùng key khác thì vòng loop đấy không nhận được giá trị (<no value>)
```
Example 2: range + helm variable
```
# values.yaml
# Flow Control: Range with List and Helm Variables
environments:
  - name: dev
  - name: qa
  - name: uat  
  - name: prod    
```
```
# Range with List
{{- range $environment := .Values.environments }}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ $environment.name }}
---  
{{- end }}           
```
Examlple 3: range with dictionary
```
values.yaml
myapps:
  config1: 
    appName: myapp1
    appType: webserver
    appTech: HTML
    appDb: mysql
  config2: 
    appName: myapp2
    appType: webserver
    appTech: HTML
    appDb: mysql
```

```
configmap.yaml
{{- $chartname := .Chart.Name }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}-configmap2
data:
{{- range $key, $value := .Values.myapps.config2 }}
{{- $key | nindent 2 }}: {{ $value }}-{{ $chartname }}
{{- end }}
```
#### named templates

Là những template hay dùng, có thể tái sử dụng. VD để định nghĩa các common label hay dùng
Cách khai báo
```
{{/* Common Labels */}}
{{- define "helmbasics.labels"}} #named template phải là unique global, nên naming convention nên là <project_name>.<relevant__template_name>
    app: nginx
    chartname: {{ .Chart.Name . }} #có template action thì khi gọi named template phải chỉ định context, nếu không giá trị sẽ là empty
{{- end }}
```

Named template có thể khai báo trực tiếp ở đầu template (cho dễ nhìn) hoặc khai trong file templates/_helpers.tpl (các file có dấu _ ở đầu sẽ không được helm tạo thành manifest)

Cách gọi named template
C1: {{ template "<named_template>" }} . VD {{ template "helmbasics.labels" }}
C2: {{ include "<named_template>" . }}. VD {{ include "helmbasics.labels" }}. Ưu điểm của C2 là có thê dùng pipeline vd  {{ include "helmbasics.labels" . | upper }}
Lưu ý nếu trong named template chỉ có key-value thì không cần chỉ định context, tuy nhiên nếu có template action thì cần chỉ định context khi gọi, nếu không sẽ bị empty. VD {{ template "helmbasics.labels" . }}

Tổng kết: Best practice khi dùng named template
- Đặt named template trong template/_helpers.tpl
- Luôn chỉ định context khi gọi named template
- Dùng include thay vì dùng template

#### printf function

Printf fucntion là 1 function quan trọng, Hàm printf trong Helm là một hàm template dùng để định dạng và in ra chuỗi theo một mẫu định dạng
syntax:
{{ printf "format string" arg1 arg2 ... }}
Ý nghĩa:

"format string" là chuỗi định dạng, có thể chứa các %d, %s, %f... để đại diện cho các kiểu dữ liệu số nguyên, chuỗi, số thực,...

arg1, arg2, ... là các giá trị sẽ được đưa vào chuỗi định dạng đó.

Một ví dụ thực tế hỏi về printf trong Helm cho thấy cách dùng để thêm ký tự "m" cho giá trị CPU units lấy từ values.yaml:

{{- printf "%dm" .Values.cpuUnits }}
Nếu cpuUnits là 1024 thì kết quả là "1024m"

ví dụ khác:
{{- printf "%s-%s" .Release.Name .Chart.Name }}
Kết quả in ra sẽ là .Release.Nam nối với .Chart.Name bằng dầu "-"

ví dụ khác
{{ prrintf "%s has %d dogs" .Name .numberDogs }}

printf rất tiện dụng khi bạn muốn kết hợp giá trị biến vào trong một chuỗi có định dạng cụ thể, ví dụ thêm đơn vị, format số, nối chuỗi có cấu trúc. Đây là hàm được dùng phổ biến trong Helm để tùy biến các giá trị cấu hình sâu sát với yêu cầu định dạng Kubernetes.

Print function kết hợp với named template
```
{{/* Kubernetes Resource Name: String Concat with Hyphen */}}
{{- define "helmbasics.resourceName" }}
{{- printf "%s-%s" .Release.Name .Chart.Name }} # %s trong hàm printf của Helm dùng để định dạng và chèn một biến hoặc giá trị ở dạng chuỗi vào chuỗi định dạng. Các ký tự định dạng khác trong printf bao gồm %d (số nguyên), %f (số thực), %v (giá trị theo định dạng mặc định)
{{- end }}
```
Cách gọi trong template
{{ include "helmbasics.resourceName" . }} #phải pass scope do ở trên dùng 2 builtin object là .Release và .Chart, (không hiểu sao không dùng được "template" mà phải dùng "include")

Các define này cũng có thể khai báo trong templates/_helper.tpl

Template trong template: để tránh duplicate code
VD: templates/_helper.tpl
```
{{/* Common Labels */}}
{{- define "helmbasics.labels"}}
    app.kubernetes.io/managed-by: helm
    app: nginx
    chartname: {{ .Chart.Name }}
    template-in-template: {{ include "helmbasics.resourceName" . }}
{{- end }}

{{/* Kubernetes Resource Name: String Concat with Hyphen */}}
{{- define "helmbasics.resourceName" }}
{{- printf "%s-%s" .Release.Name .Chart.Name }}
{{- end }}
```
#### helm chart dependency
là các thành phần hoặc charts khác mà một chart cần có để hoạt động 
Trong Helm, dependencies của một chart được khai báo trong phần dependencies của file Chart.yaml. Các dependencies này sẽ được Helm tự động tải về và quản lý bên trong thư mục charts/ của chart chính khi thực hiện các lệnh như helm dependency update, giúp bạn dễ dàng tích hợp các dịch vụ bên ngoài hoặc các thành phần phụ trợ cần thiết cho ứng dụng, ví dụ như Redis, MySQL

`copy từ trên xuống`

b. chart.yaml: thông tin metadata về chart (tên, phiên bản, phiên bản của application (thường là docker image tag), mô tả, dependencies)
Trường dependencies trong file Chart.yaml dùng để khai báo các chart khác mà chart hiện tại phụ thuộc vào. Đây là cách để bạn xác định “subchart” hoặc “dependency chart” mà ứng dụng của bạn cần, giúp tự động hóa quá trình cài đặt và đảm bảo đầy đủ các thành phần cần thiết khi triển khai ứng dụng trên Kubernetes
Lợi ích và ứng dụng
Tự động hóa: Triển khai đầy đủ các thành phần phức hợp (ví dụ: ứng dụng chính + database, redis, prometheus…) chỉ với một chart cha.

Quản lý version: Dễ dàng kiểm soát phiên bản chart phụ thuộc bằng trường version.

Tái sử dụng: Dùng lại các chart phổ biến từ cộng đồng hoặc nội bộ như một module nhỏ.

VD
```
dependencies:
  - name: mysql #phải đúng tên chart trong bitnami repo
    version: "9.3.4"
    repository: "https://charts.bitnami.com/bitnami" #hoặc có thể thay url bằng @<tên_repo_trên_local>. VD repository: @myrepo . Cách này ko recommended do không thể share giữa các máy
  - name: redis
    version: "~14.0.0"
    repository: "https://charts.bitnami.com/bitnami"
```
Lưu ý version của dependency có thể khai báo động, không cần fix cứng
VD
version: "= 9.10.8"  -> fix cứng
version: "!= 9.10.8" -> không dùng version này
version: ">= 9.10.8" -> chỉ dùng version lớn hơn 9.10.8
version: "<= 9.10.8"
version: "> 9.10.8"   
version: "< 9.10.8"
version: ">= 9.10.8 < 9.11.0"  
version: ^9.10.1  -> major bắt buộc là 9, minor phải lớn  hơn 10 và patch lớn hơn 1 (is equivalent to >= 9.10.1, < 10.0.0)
version: ^9.10.x  -> major bắt buộc là 9, minor phải lớn  hơn 10 và patch sao cũng được (is equivalent to >= 9.10.0, < 10.0.0   
version: ^9.10    is equivalent to >= 9.10, < 10
version: ^9.x     is equivalent to >= 9.0.0, < 10        
version: ^0       is equivalent to >= 0.0.0, < 1.0.0
version: ~9.10.1  -> major và minor bằng 9 và 10, lấy bất kỳ patch nào (is equivalent to >= 9.10.1, < 9.11.0 # Patch-level version match)
version: ~9.10    is equivalent to >= 9.10, < 9.11
version: ~9       is equivalent to >= 9, < 10
version: ^9.x     is equivalent to >= 9.0.0, < 10        
version: ^0       is equivalent to >= 0.0.0, < 1.0.0

e. Thư mục charts/: chứa các chart khác mà chart hiện tại cần dùng, để ở dạng <name>.tgz. Là nơi CHỨA các package/chart phụ đã được tải về, ta có thể tải các file .tgz thủ công về sau đó copy vào thư mục này (hữu dụng khi dùng cho môi trường không có Internet hoặc cần cố định phiên bản). Đây cũng là nơi chứa dependency được khai báo trong trường dependencies của file Chart.yaml sẽ được Helm tự động tải về và lưu trong thư mục charts/ khi bạn chạy lệnh helm dependency update

Các lệnh làm việc với dependency:
21. helm dependency list -> xem các dependencies khai báo trong file chart.yaml (lưu ý cần đứng trong root directory của chart để chạy lệnh này) (lưu ý là chi list ra chứ không thật sự download)
22. helm dependency build -> rebuild thư mục charts/ dựa trên file chart.lock
23. helm dependency update -> khi update các dependencies khai báo trong file chart.yaml thì ta chạy lệnh này sẽ sinh ra file <name-version>.tgz mới trong thư mục charts
Chatgpt: Lệnh này sẽ tự động tải về các chart phụ thuộc và lưu vào thư mục charts/ của chart chính, đồng thời tạo file Chart.lock (là file chung nằm ở root) chứa thông tin chính xác về phiên bản thực tế của từng chart phụ thuộc

Điểm khác nhau: helm dep update làm việc với file chart.yaml, helm dep build làm việc với file chart.lock. Nếu ta thay đổi file chart.yaml mà không thay đổi file chart.lock thì khi chạy lệnh helm dep build sẽ báo lỗi "chart.lock không đồng bộ với chart.yaml". 
nếu ta xóa file chart.lock đi thì lệnh helm dep build sẽ đọc vào trở thành lệnh helm dep update

---
helm dependency alias: useful khi cần add 1 chart nhiều lần vào file chart.yaml
Chatgpt: alias trong phần khai báo dependencies của Chart.yaml đóng vai trò đặt tên "bí danh" (alias) cho các chart phụ thuộc khi bạn muốn sử dụng cùng một chart nhiều lần trong cùng một chart cha. Dù cùng trỏ đến cùng một chart (mychart4, version 0.1.0), nhờ alias mà khi Helm triển khai, mỗi instance sẽ được nhận diện bằng tên alias riêng biệt, đồng thời bạn có thể cấu hình giá trị (values) khác nhau cho từng instance.
VD:
```
apiVersion: v2
name: parentchart
description: Learn Helm Dependency Concepts
type: application
version: 0.1.0
appVersion: "1.16.0"
dependencies:
- name: mychart4 #cần dùng alias do chart name không thể giống nhau
  version: "0.1.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart4dev
- name: mychart4
  version: "0.1.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart4qa  #mychart4 xuất hiện hai lần, nhưng alias khác nhau là childchart4dev và childchart4qa. Điều này giúp bạn deploy ra hai dịch vụ cùng kiểu nhưng gán cho mỗi dịch vụ một bộ tên cấu hình và management riêng biệt trong Helm. Lưu ý khi chạy helm dep update thì chỉ tạo ra 1 file mychart4.tgz trong thư mục charts/ dù cho có khai báo nhiều lần
- name: mychart2
  version: "0.4.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart2
```
---

helm dependency condition: dùng để for enabling or disabling Sub Charts or Child Charts
Cách làm: define 1 trường trong values.yaml để enable/disable chart
```
#values.yaml
mychart4:
  enabled: false
mychart2:
  enabled: true
```
```
dependencies:
- name: mychart4
  version: "0.1.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart4
  condition: mychart4.enabled #đặt là gì cũng được, chỉ cần trả về true (chart được enable) hoặc false (chart bị disable). Lưu ý nếu không khai báo trong values.yaml thì mặc định chart sẽ được enable -> tóm lại: mặc định các chart khai trong dependency sẽ được enable, nếu muốn disable phải khai trong values.yaml
- name: mychart2
  version: "0.4.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart2
  condition: mychart2.enabled
```
---
denpendency + tags
Tag dùng để group các chart lại và enable hoặc disable cùng nhau, hữu ích nếu có nhiều sub chart
```
dependencies:
- name: mychart4
  version: "0.1.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart4dev
  #condition: childchart4dev.enabled
  tags: 
    - frontend #mychart4 sẽ thuộc group frontend
- name: mychart2
  version: "0.4.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  alias: childchart2
  #condition: childchart2.enabled
  tags: 
    - backend #mychart4 sẽ thuộc group backend
```
Để disable hoặc enable 1 group, ta khai báo trong values.yaml
```
tags:
  frontend: false
  backend: false
```
---
helm dependency - overriding sub chart value from parent chart
Cách để pass value cho sub chart từ file values.yaml của parent chart
Điều kiện: phải khai báo condition trong dependency
```
apiVersion: v2
name: parentchart
description: Learn Helm Dependency Concepts
type: application
version: 0.1.0
appVersion: "1.16.0"
dependencies:
- name: mychart4
  version: "0.1.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  condition: mychart4.enabled
- name: mychart2
  version: "0.4.0"
  repository: "https://stacksimplify.github.io/helm-charts/"
  condition: mychart2.enabled
  alias: childchart2
```
trong file value.yaml, tất cả những giá trị dưới enabled sẽ được truyền cho sub chart
```
mychart4: 
  enabled: true
  replicaCount: 3
childchart2: #lưu ý nếu trong dependency có sử dụng alias thì ở đây phải khai báo bằng alias, dùng chart name sẽ không được
  enabled: true  
  replicaCount: 3
  image:
    repository: nginx
```
---
Global value
- Là value dùng được cho cả parent chart và sub chart. Ta define global value trong parent chart values.yaml và dùng trong sub chart
- Cách thực hiện: do muốn dùng global value nên trong folder charts/ không phải là file .tgz mà là thư mục có cấu trúc của 1 chart -> cần thực hiện download file tgz về thư mục charts/ và giải nén ra bằng lệnh
- Tại sao cần untar sub chart: lý do là để có thể custom các chart được build sẵn cho phù hợp với nhu cầu
```
helm pull https://stacksimplify.github.io/helm-charts/mychart4-0.1.0.tgz --untar
```
dependency của chart.yaml cần trỏ về thư mục
```
apiVersion: v2
name: parentchart
description: Learn Helm Dependency Concepts
type: application
version: 0.1.0
appVersion: "1.16.0"
dependencies:
- name: mychart4
  version: "0.1.0"
  repository: "file://charts/mychart4"
  alias: childchart4
  tags: 
    - frontend
- name: mychart2
  version: "0.4.0"
  repository: "file://charts/mychart2"
  alias: childchart2
  tags: 
    - backend
```
Trong file values.yaml của parent chart khai báo global value
```
#parentchart/values.yaml
global:
  replicaCount: 4
```
Sau này khi muốn gọi global value thì dùng cú pháp {{ .Values.global.replicaCount }} thay vì {{ .Values.replicaCount }} (áp dụng cho cả file values.yaml của parent chart và sub chart)

---
helm dependency import value
Dùng để export value từ file values.yaml của child chart và import value đấy vào values.yaml của parent chart, từ đó template của parent chart có thể access các giá trị của child chart
(không hiểu tại sao cần phải access vào giá trị trong file values.yaml của child chart)
a. import value explicit
Đầu tiên cần export value từ chart con ra
```
#charts/mychart1/values.yaml
exports:
  mychart1Data:
    mychart1appInfo:
      appName: kapp1
      appType: MicroService
      appDescription: Used for listing products
```
Sau đó import vào trong parent chart thông qua file Chart.yaml
```
- name: mychart1
  version: "0.1.0"
  repository: "file://charts/mychart1"
  alias: childchart1
  tags: 
    - frontend
  import-values:
    - mychart1Data # Explicit Values Import Usecase
```

Từ giờ ta đã có thể dùng các value này trong parent chart. Lưu ý quan trọng: `sau khi đã import vào chart cha thì ta gọi trực tiếp "key" để lấy giá trị chứ không sử dụng mapping name`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name:  {{ include "parentchart.fullname" . }}-import-explicit
data:
{{- toYaml .Values.mychart1appInfo | nindent 2 }}
```
b. import value implicit
Cách này thì không cần define export explicit trong file values.yaml của sub chart 

```
#Chart.yaml
- name: mychart2
  version: "0.4.0"
  repository: "file://charts/mychart2"
  alias: childchart2
  tags: 
    - backend
  import-values: # Implicit Values Usecase
    - child: service  #đây là key trong file values.yaml của child chart
      parent: mychart2service  #đây là mapping name, parent chart sẽ access giá trị từ child chart bằng name này
    - child: image 
      parent: mychart2image
```
Sau đó gọi trong template của parent chart
```
apiVersion: v1
kind: ConfigMap
metadata:
  name:  {{ include "parentchart.fullname" . }}-import-implicit
data:
  serviceType: {{ .Values.mychart2service.type }} #phải access thông qua mapping name
  servicePort: {{ .Values.mychart2service.port | quote}}
  servicenodePort: {{ .Values.mychart2service.nodePort | quote }}
  imageRepository: {{ .Values.mychart2image.repository }}
```
Điểm khác nhau khi access vào giá trị của chart con bằng 2 cách:
- Khi explicit import thì access trực tiếp vào key, không qua mapping name
- Khi implicit import thì phải access thông qua mapping name
CHÁN R NGHỈ THÔI
