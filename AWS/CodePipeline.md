Codebuild


```
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.12
      docker: 20
    
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
      
artifacts:
  files:
    - python-web-task.json
    - imagedefinitions.json

# Cache Docker layers
cache:
  paths:
    - '/var/lib/docker/**'

```

- Các tên phase như install, pre_build, build, post_build là quy ước chuẩn của AWS CodeBuild. AWS định nghĩa chính xác 4 phase này trong buildspec.yaml theo thứ tự thực thi cố định.​

- artifacts định nghĩa file nào được lưu trữ vào S3 sau build để dùng cho pipeline tiếp theo (CodePipeline/ECS). Nếu không khai báo = không có artifact output
  - files: List file/folder cần lưu

- cache tăng tốc build bằng cách lưu Docker layers giữa các build. Build lần 2 sẽ nhanh hơn vì không pull lại layers
  - paths: Đường dẫn cache (/var/lib/docker/** cho Docker layers)
