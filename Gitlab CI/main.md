
GitLab chỉ đọc file .gitlab-ci.yml ở root directory làm config chính. Pipeline chỉ trigger (được tạo) khi GitLab detect file .gitlab-ci.yml (hoặc tương đương) ở root directory của repo. File ở subdirectory không tự động trigger pipeline

Nếu ở root không có file .gitlab-ci.yml mà chỉ tạo .gitlab-ci.yml trong subdirectory thì khi push thay đổi code trong thư mục đó: pipeline KHÔNG trigger.
​Nguyên nhân là do GitLab quét root → không thấy config → coi repo không có CI/CD → bỏ qua push, không tạo pipeline.
​File subdirectory không được tự động đọc như config chính; chỉ dùng qua include hoặc child pipelines từ root


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

---

Các trường before_script, variables, tags, rules, và artifacts là những trường phổ biến nhất ngoài services và id_tokens trong GitLab CI/CD.
​

before_script & after_script
- before_script: Chạy trước script chính (setup env, install tools)
- after_script: Chạy sau script (cleanup, notifications)

```
before_script:
  - apk add aws-cli docker    # Install tools
script:
  - aws sts get-caller-identity
after_script:
  - docker system prune -f    # Cleanup
```

variables
- Global/job vars override mặc định, inject env vars
- Secrets từ GitLab CI/CD variables (protected/masked)

```
variables:
  AWS_REGION: "ap-northeast-1"
  DOCKER_IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

tags & needs
- tags: Chọn GitLab Runner cụ thể (docker, kubernetes executor)
- needs: Chạy parallel/thuộc jobs khác (bỏ stage dependency)

```
build:
  tags:
    - docker-runner           # Chọn runner có Docker dind
deploy:
  needs: [build]              # Chạy ngay sau build, không đợi stage
```

rules & artifacts
- rules: Điều kiện chạy job (if branch, changes files)
- artifacts: Lưu file giữa jobs/stages (Docker images, reports)
```
rules:
  - if: $CI_COMMIT_BRANCH == "main"
artifacts:
  paths:
    - dist/
  expire_in: 1 week
```


---
Giải thích rõ hơn về before_script

before_script chạy luôn luôn trước script, thường dùng để chạy các lệnh setup môi trường trước script chính, giúp install tools, config auth, hoặc load env vars.

Cách sử dụng cơ bản, có 2 cách:
- Đặt ở global (default cho tất cả jobs)
- Đặt trong mỗi job

```
default:
  before_script:
    - apk update               # Global setup

job:
  before_script:
    - echo "Job-specific setup"
  script:
    - echo "Main task"
```

Lưu ý nếu define cả global và trong job thì before_script của job sẽ ghi đè global (không merge). Ví dụ
```
default:
  before_script:
    - apk add --no-cache jq    # Global: jq cho tất cả jobs

deploy-ecr:
  image: docker:stable
  before_script:              # Override: AWS + Docker specific
    - apk add --no-cache aws-cli docker-cli
    - aws configure set region ap-northeast-1
  script:
    - aws sts get-caller-identity
    - docker build .

test-unit:
  before_script: []           # Skip global, chỉ Python test
  image: python:3.12
  script:
    - pip install -r requirements.txt
    - pytest
```

Để kế thừa (merge) global thì dùng !reference (từ GitLab 13.7+)
```
deploy-ecr:
  before_script:
    - !reference [.default, before_script]  # Kế thừa global
    - apk add aws-cli                      # Thêm custom
```
