Codebuild


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
​
