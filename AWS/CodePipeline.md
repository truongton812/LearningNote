## Codebuild


```
version: 0.2

# Các biến môi trường, secrets, shell
env:
  variables:
    KEY: "value"
  secrets-manager:
    SECRET_KEY: "my-secret-key"

phases:
  install:
    runtime-versions: #để chọn phiên bản Node.js (hoặc Java, Python, Go…) theo build image bạn dùng
      python: 3.12
      docker: 20
    commands:
      - echo "Installing dependencies"
      - npm ci
    
  pre_build:
    commands:
      - echo Đăng nhập vào các container registries...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPOSITORY_URI
      
  build:
    commands:
      - echo Build bắt đầu $(date)
      - echo Build Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $ECR_REPOSITORY_URI:$IMAGE_TAG
      
  post_build:
    commands:
      - echo Push image lên ECR...
      - docker push $ECR_REPOSITORY_URI:$IMAGE_TAG
      - echo Ghi task definition...
      - taskDef='python-web-task.json'
      - envsubst < task-definition.json > $taskDef
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $ECR_REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
      
artifacts: # cho CodeBuild biết những file nào sẽ được upload lên S3 hoặc dùng cho stage tiếp theo (ví dụ trong CodePipeline).
  files:
    - python-web-task.json
    - imagedefinitions.json
  base-directory: .
  discard-paths: yes
  name: "DeploymentArtifacts" #Tên được CodePipeline dùng để làm Input Artifact cho các stage tiếp theo.


# Cache để tăng tốc giữa các build
cache:
  paths:
    - '/var/lib/docker/**'

```

Trong buildspec.yml của AWS CodeBuild, khối env: (version 0.2) có thể chứa đầy đủ các field sau.
​
- env: shell: chọn shell chạy lệnh; Linux hỗ trợ bash, /bin/sh, Windows hỗ trợ powershell.exe, cmd.exe.
​- env: variables: khai báo env var dạng plain text theo cặp key: value.
​- env: parameter-store: map key (tên env var trong build) → value (tên/đường dẫn parameter trong SSM Parameter Store).
​- env: secrets-manager: map key (tên env var trong build) → tham chiếu secret theo mẫu <secret-id>:<json-key>:<version-stage>:<version-id> (các phần sau secret-id là optional tuỳ cách lấy).
​- env: exported-variables: list các tên biến muốn export cho stage sau của CodePipeline (biến phải tồn tại trong container trong lúc build).
​- env: git-credential-helper: yes | no để bật/tắt Git credential helper của CodeBuild (không hỗ trợ một số trường hợp như webhook public repo). 
​
Ví dụ
```
version: 0.2
env:
  shell: bash
  variables:
    APP_ENV: "stg"
  parameter-store:
    DB_USER: "/myapp/db/user"
  secrets-manager:
    DB_PASS: "mysecret:password"
  exported-variables:
    - APP_ENV
  git-credential-helper: yes
```


---
Trường env trong buildspec.yml (AWS CodeBuild) dùng để khai báo và lấy biến môi trường cho quá trình build, giúp bạn cấu hình hành vi build theo môi trường (dev/stg/prod) hoặc truyền các tham số/secret vào các lệnh trong phases.
​

env nhận những “giá trị” nào?
Trong buildspec.yml (version 0.2), env thường có các nhóm cấu hình chính sau:

env: variables: các cặp key/value dạng chuỗi (ví dụ MY_VARIABLE: "my-value").
​

env: parameter-store: map từ tên biến môi trường sang đường dẫn Parameter Store (SSM) để CodeBuild nạp giá trị về và expose thành env var trong build.
​

env: exported-variables: danh sách tên biến mà CodeBuild “export” ra để các bước sau (ví dụ CodePipeline action downstream) có thể nhận lại.
​

Ví dụ tối giản:

text
version: 0.2
env:
  variables:
    MY_VARIABLE: "my-value"
  parameter-store:
    DB_CONNECTION_STRING: /myapp/database/connectionstring
  exported-variables:
    - MY_VARIABLE
phases:
  build:
    commands:
      - echo $MY_VARIABLE
(Ý nghĩa từng khối như mô tả ở trên.)
​

Trường hợp đặc biệt: build matrix (batch build)
Nếu bạn dùng batch build / build matrix, trong phần batch.build-matrix...env.variables bạn có thể khai báo biến theo dạng danh sách nhiều giá trị (để chạy nhiều job với các giá trị khác nhau), ví dụ MY_VAR: [VALUE1, VALUE2, VALUE3].
​

Lưu ý nhanh về kiểu dữ liệu
Các giá trị dưới env.variables về bản chất là giá trị cấu hình dạng text trong YAML (thường bạn để string, có/không có dấu ngoặc kép), và khi vào build thì được dùng như biến môi trường trong shell (tham chiếu bằng $VAR).
​

Nếu bạn muốn, bạn dán đoạn buildspec.yml của bạn (hoặc mục tiêu: muốn lấy secret từ đâu, muốn export biến nào), mình sẽ chỉ ra cấu trúc env phù hợp và ví dụ hoàn chỉnh.

Prepared using GPT-5.2
có env: secret-manager ko
Có. Trong buildspec.yml của AWS CodeBuild, dưới env có mục secrets-manager để map biến môi trường cục bộ sang secret trong AWS Secrets Manager.
​

Cú pháp
Dạng tổng quát (theo Buildspec reference):

env:

secrets-manager:

<LOCAL_ENV_NAME>: <secret-id>:<json-key>:<version-stage>:<version-id>
​

Trong đó:

secret-id có thể là tên hoặc ARN của secret.
​

json-key, version-stage, version-id là phần tuỳ chọn (dùng khi secret là JSON và/hoặc muốn chỉ định version).
​

Ví dụ
Lấy 1 key từ secret (secret chứa key-value), gán vào biến môi trường LOCAL_SECRET_VAR:

text
version: 0.2
env:
  secrets-manager:
    LOCAL_SECRET_VAR: "TestSecret:MY_SECRET_VAR"
phases:
  build:
    commands:
      - echo "$LOCAL_SECRET_VAR"
Ví dụ này được AWS nêu trực tiếp trong tài liệu (TestSecret là secret, MY_SECRET_VAR là key; LOCAL_SECRET_VAR là tên env var trong build).
​

Lưu ý quan trọng
IAM role của CodeBuild project phải có quyền đọc secret tương ứng (ví dụ secretsmanager:GetSecretValue), nếu không build sẽ fail khi resolve biến.
​


---



- Các tên phase như install, pre_build, build, post_build là quy ước chuẩn của AWS CodeBuild. AWS định nghĩa chính xác 4 phase này trong buildspec.yaml theo thứ tự thực thi cố định.​

- Phase install có tác dụng cài đặt môi trường runtime (Python, Docker, Node.js...) và chạy các lệnh setup ban đầu trước khi build. Ví dụ:​

```
phases:
  install:
    runtime-versions:
      python: 3.12
      docker: 20
```
Chọn phiên bản runtime, quyết định image base AWS dùng: AWS tải đúng image chứa Python 3.12, Docker 20

- artifacts định nghĩa file nào được lưu trữ vào S3 sau build để dùng cho pipeline tiếp theo (CodePipeline/ECS). Nếu không khai báo = không có artifact output
  - files: List file/folder cần lưu

- cache tăng tốc build bằng cách lưu Docker layers giữa các build. Build lần 2 sẽ nhanh hơn vì không pull lại layers
  - paths: Đường dẫn cache (/var/lib/docker/** cho Docker layers)
 
---

- Khi xây dựng pipeline với CodeBuild để build và CodeDeploy để deploy lên ECS thì trong artifact của CodeBuild cần xuất ra 1 file đặc biệt là imagedefinitions.json (thường dùng printf ra JSON). File này dùng để hướng dẫn cho CodeDeploy biết nên dùng image Docker nào để deploy lên ECS.

- Nội dung file imagedefinitions.json:

```json
[
  {
    "name": "my-container",
    "imageUri": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/my-app:latest"
  }
]
```

- Tác dụng chính của imagedefinition.json: map giữa container name trong ECS task definition với image URI thực tế trên ECR (CodePipeline sẽ đọc file imagedefinitions.json, tìm đến container name được khai báo trong imagedefinitions.json và sửa image URI của container đấy). Nếu thiếu file này (hoặc tên sai, format sai), stage deploy ECS trong CodePipeline sẽ báo lỗi "Did not find the image definition file imagedefinitions.json"

---

Artifact trong AWS CodeBuild là tập hợp các file đầu ra mà bạn muốn lưu lại sau khi build xong (ví dụ: bundle frontend, file JAR, Docker image, hoặc các file cấu hình như imagedefinitions.json). Artifact này thường được lưu trong S3 và có thể dùng cho các bước tiếp theo như CodeDeploy / ECS / S3 deploy trong CodePipeline.

Cách khai báo Artifact trong buildspec.yaml 
```
artifacts:
  files: #danh sách các file hoặc pattern được đưa vào artifact 
    - "dist/**/*"
    - "package.json"
  base-directory: . #thư mục gốc để tính tương đối cho các đường dẫn trong files.
  discard-paths: yes #yes sẽ bỏ các thư mục trung gian trong artifact (ví dụ: chỉ lưu file, không giữ cấu trúc folder).
  name: "DeploymentArtifacts"

```

Trên giao diện CodeBuild cũng có khai báo Artifact với 2 lựa chọn là `No artifacts` và `S3`
- Nếu chọn `No artifacts` : sẽ không lưu artifact vào đâu cả
- Nếu chọn `S3`: chỉ định S3 bucket để chứa artifact, cần khai báo: Bucket, Name (tên S3 object), Path (prefix trong bucket). Nếu không khai báo thì CodePipeline sẽ tự tạo bucket để quản lý

→ Có thể nói:

artifacts trong buildspec.yaml quy định “nội dung” của artifact (những file gì).

Artifacts trên giao diện CodeBuild quy định “địa chỉ lưu” artifact (lưu ở đâu trong S3).

Hai chỗ này bổ sung cho nhau, không phải “giống nhau hoàn toàn”. Nếu không khai báo artifacts trong buildspec.yaml, CodeBuild sẽ không upload file nào cả dù bạn đã cấu hình bucket trên GUI.

Nếu trong buildspec.yaml có khai báo artifacts nhưng trên GUI chọn là `No artifacts` thì CodeBuild sẽ bỏ qua, không tạo artifact nào. Khi build xong sẽ không thấy file nào được upload lên S3 và cũng không có artifact nào để dùng trong CodePipeline (stage deploy sẽ không tìm thấy input artifact).

---

AWS CodeBuild artifacts là các tệp đầu ra được tạo ra sau quá trình build trong AWS CodeBuild. Artifacts có các loại chính: S3 (lưu vào bucket cụ thể với path, name, namespaceType như BUILD_ID), CODEPIPELINE (tích hợp với CodePipeline), hoặc NO_ARTIFACTS (không tạo output).
​

Type CODEPIPELINE trong AWS CodeBuild là loại artifact được thiết kế đặc biệt để tích hợp với AWS CodePipeline, giúp tự động quản lý việc lưu trữ và truyền artifacts giữa các stage trong pipeline CI/CD. Khi CodeBuild project được cấu hình artifacts: type: CODEPIPELINE, CodeBuild không cần chỉ định S3 bucket cụ thể vì CodePipeline sẽ tự động xử lý việc upload/download artifacts vào bucket artifact của pipeline (thường tên codepipeline-<region>-<account>). Artifacts được zip tự động và truyền trực tiếp làm input cho stage tiếp theo (như Deploy), đảm bảo tính liền mạch mà không cần can thiệp thủ công.
​
Lưu ý: dùng type CODEPIPELINE khi CodeBuild là action trong CodePipeline; nếu chạy standalone, chuyển sang S3.
​​
---

AWS CodeBuild mặc định không nằm trong VPC; nó chạy trong môi trường build được quản lý bởi AWS, lúc này CodeBuild không thể truy cập trực tiếp các tài nguyên trong VPC (ví dụ RDS, ElastiCache, private ECR, EC2 private…), vì không có route từ CodeBuild vào VPC.

Có thể attach CodeBuild vào VPC bằng cách enable VPC access (cần chỉ định VPC ID, Subnet (thường là private subnet) và Security group). Khi bật VPC access, CodeBuild sẽ run build trong network của VPC đó → có thể truy cập các tài nguyên trong VPC (RDS, private ALB, Redis, etc.).

---


CODEBUILD_SOURCE_VERSION và CODEBUILD_RESOLVED_SOURCE_VERSION đều là biến môi trường của AWS CodeBuild, nhưng mục đích và nội dung khác nhau
- CODEBUILD_SOURCE_VERSION là “input version” bạn truyền cho CodeBuild.
- CODEBUILD_RESOLVED_SOURCE_VERSION là “version đã được resolve” xong, thường là một giá trị ổn định hơn để dùng làm tag, version, v.v.

1. CODEBUILD_SOURCE_VERSION

Biến này phản ánh cách bạn chỉ định source version khi khởi tạo build:

- Với CodeCommit, GitHub, GitHub Enterprise Server, Bitbucket có thể là: commit ID (SHA), branch name (ví dụ: main, feature/login), tag name (ví dụ: v1.0.0), hoặc pr/pull-request-number (với webhook pull request event trên GitHub).
- Với Amazon S3 là version ID của object chứa input artifact (file ZIP, tar…).

Ví dụ: Nếu bạn cấu hình build trên GitHub với branch thì CODEBUILD_SOURCE_VERSION = refs/heads/main. Nếu bạn cấu hình build cho commit cụ thể thì CODEBUILD_SOURCE_VERSION = 1234567890abcdef1234567890abcdef12345678. Nếu là pull request thì CODEBUILD_SOURCE_VERSION = pr/123

2. CODEBUILD_RESOLVED_SOURCE_VERSION
Biến này là version sau khi CodeBuild đã resolve source, tức là:
- Với CodeCommit, GitHub, GitHub Enterprise Server, Bitbucket: Luôn là commit ID (SHA), ngay cả khi bạn nhập branch hay tag ban đầu.
- Với CodePipeline: Là source revision mà CodePipeline cung cấp (thường là commit SHA).
- Nếu CodePipeline không resolve được (ví dụ S3 không bật versioning), biến này không được set.


Nếu bạn muốn tag Docker image bằng commit SHA thực tế thì nên dùng biến CODEBUILD_RESOLVED_SOURCE_VERSION vì nó luôn là SHA (nếu có), rất hợp lý cho Docker tag.

Lưu ý quan trọng khi dùng: CODEBUILD_RESOLVED_SOURCE_VERSION chỉ có sẵn sau phase DOWNLOAD_SOURCE, nên nếu bạn dùng trong phases: build thì ok, nhưng nếu có script chạy quá sớm thì có thể thấy null / không set. Nếu bạn đang dùng S3 (không phải Git) và không bật versioning, có thể biến CODEBUILD_RESOLVED_SOURCE_VERSION không tồn tại, lúc đó bạn buộc phải dùng CODEBUILD_SOURCE_VERSION hoặc logic khác.

---

## CodePipeline

Stage Source trong AWS CodePipeline output ra một artifact chứa mã nguồn ứng dụng dưới dạng file ZIP. Source artifact thường tên mặc định như "SourceArtifacts", truyền trực tiếp vào Build stage để compile code.
​
Artifact lưu trữ trong S3 bucket của CodePipeline (ví dụ: codepipeline-{region}-{random}).
​
