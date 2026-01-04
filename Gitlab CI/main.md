Mẫu gitlab-ci.yaml
```
stages:
  - build           # Nhóm 1: Tất cả job thuộc build chạy parallel
  - test            # Nhóm 2: Chờ build xong mới chạy
  - deploy          # Nhóm 3: Chờ test xong mới chạy
build-ecr:
  stage: build            # Thuộc stage build
  script:
    - docker build .
    - docker push ecr...

deploy-staging:
  stage: deploy-staging   # Thuộc stage deploy-staging
  needs: [build-ecr]      # Chờ build job cụ thể (không đợi cả stage)

deploy-job:
  image: docker:stable
  services:
    - docker:dind
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  variables:
    ROLE_ARN: "arn:aws:iam::060795929133:role/gitlabrole"
    DOCKER_TLS_CERTDIR: "/certs"  # Bỏ qua TLS cho dind
  before_script:
    - apk add --no-cache aws-cli  # Install AWS CLI v2
  script:
    - |
      aws_sts_output=$(aws sts assume-role-with-web-identity \
        --role-arn "${ROLE_ARN}" \
        --role-session-name "test" \
        --web-identity-token $GITLAB_OIDC_TOKEN \
        --duration-seconds 3600 \
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
        --output text)
    - |
      export AWS_ACCESS_KEY_ID=$(echo "$aws_sts_output" | cut -d$'	' -f1)
      export AWS_SECRET_ACCESS_KEY=$(echo "$aws_sts_output" | cut -d$'	' -f2)
      export AWS_SESSION_TOKEN=$(echo "$aws_sts_output" | cut -d$'	' -f3)
    - aws sts get-caller-identity
    - aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 060795929133.dkr.ecr.ap-northeast-1.amazonaws.com
    - docker build -t cg-utils/dot-game .
    - docker tag cg-utils/dot-game:latest 060795929133.dkr.ecr.ap-northeast-1.amazonaws.com/cg-utils/dot-game:latest
    - docker push 060795929133.dkr.ecr.ap-northeast-1.amazonaws.com/cg-utils/dot-game:latest
```
Job là đơn vị thực thi nhỏ nhất. jobs là các công việc cụ thể chạy trong stage. Có thể multiple jobs cùng stage

stages định nghĩa các giai đoạn thực thi tuần tự trong pipeline GitLab CI/CD.
- Chạy tuần tự: Stage sau chờ stage trước hoàn thành
- Parallel trong stage: Các job cùng stage chạy song song

GitLab CI tự động áp dụng 3 stages mặc định build, test, deploy nếu không khai báo stages trong .gitlab-ci.yml. Trong job ta khai báo stage build/test/deploy để chỉ định stage mà job thuộc về. Nếu không khai báo thì sẽ mặc định thuộc stage test


---
Trường services dùng để khai báo các service containers chạy cùng job chính (main container từ image)

Mỗi service có network riêng, job chính tự động connect qua Docker network (localhost)

Ví dụ services: - docker:dind: Chạy Docker daemon để job chính dùng lệnh docker build/push

Service dùng chung variables từ job, có thể pass thêm vars riêng
```
job:
  image: docker:stable          # Main container (có docker CLI)
  services:
    - docker:dind              # Service container (Docker daemon)
  script:
    - docker build .           # Main container gọi service qua tcp://docker:2376
```
Trường id_tokens dùng để tạo OIDC JWT tokens tự động từ GitLab (không cần AWS keys)

Dùng cho aws sts assume-role-with-web-identity để lấy temp AWS credentials

aud: https://gitlab.com: Audience xác định GitLab là OIDC provider

```
id_tokens:
  GITLAB_OIDC_TOKEN:           # Tên biến chứa JWT token
    aud: https://gitlab.com    # GitLab issuer URL
```
---

Khác nhau giữa dùng dấu `-` và `|` khi multiline: dấu `-` tạo array các lệnh riêng biệt, dấu `|` tạo block multiline một lệnh duy nhất.

Dấu - (List/Array)

```
script:
  - apk add aws-cli          # Lệnh 1
  - aws sts get-caller-identity  # Lệnh 2  
  - docker build .           # Lệnh 3
Kết quả: 3 lệnh riêng, job fail nếu bất kỳ lệnh nào fail
```

​

Dấu | (Literal block)
```
script:
  - |
    apk add aws-cli
    aws sts get-caller-identity
    docker build .
Kết quả: 1 lệnh duy nhất, chỉ fail nếu lệnh cuối fail
```
​

Kinh nghiệm: Luôn dùng - cho setup env để đảm bảo từng bước thành công trước khi chạy tiếp
