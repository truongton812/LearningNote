
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
### 1. SCOPE CỦA VARIABLES

các biến trong GitLab CI không tự động persist giữa các job/stage. Đặc điểmL
- Mỗi job chạy trong container/environment riêng biệt, nên biến chỉ có scope cục bộ. Mỗi job luôn chạy trong container riêng biệt, bất kể có cùng stage. Cùng stage chỉ có nghĩa chạy song song.

- Mỗi script (VD before_script, script, after_script) chạy trong 1 shell mới

- GitLab CI mặc định tạo lại container/shell mới cho mỗi job/stage, do đó môi trường và config không được giữ nguyên. Để chia sẻ dữ liệu hoặc môi trường giữa các job/stage thì phải dùng artifact hoặc cache

 
Scope của biến

1. Predefined Variables (Toàn cục)

Các biến built-in như $CI_COMMIT_SHA, $CI_JOB_NAME, $CI_PIPELINE_ID available cho tất cả job trong pipeline.
​

2. Variables trong .gitlab-ci.yml
```
variables:                    # Scope: TOÀN BỘ PIPELINE (mọi job)
  GLOBAL_VAR: "global"

job1:
  variables:                  # Scope: CHỈ job1
    LOCAL_VAR: "local"
  script:
    - echo $GLOBAL_VAR      # OK
    - echo $LOCAL_VAR       # OK
```

3. Script Variables (Cục bộ nhất)
```
script:
  - export MY_VAR="hello"    # Chỉ trong script của job này, không truyền sang job khác hoặc script khác (như before_script, after_script)
MY_VAR chỉ tồn tại trong job đó, không truyền sang job khác.
```

### 2. Cách persist biến giữa các job

#### 2.1 Dùng .env artifacts - Best practice:

`reports: dotenv` tự động load biến từ file .env vào environment của job sau

Cách hoạt động dotenv artifacts
```
Job A tạo file → artifacts lưu trữ → Job B download → GitLab parse .env → inject biến
```

Cú pháp
```
build-job:
  stage: build
  script:
    - echo "DB_URL=postgres://user:pass@localhost" > app.env
    - echo "BUILD_TIME=$(date)" >> app.env
  artifacts:
    reports:                    # reports là key bắt buộc
      dotenv: app.env           # Tên file tùy ý (.env, build.env...)

```
Job sau tự động có biến:
```
deploy-job:
  stage: deploy
  needs: ["build-job"]          # Quan trọng: phải needs để nhận artifacts
  script:
    - echo "Connecting to $DB_URL"     # ✅ Có sẵn
    - echo "Built at $BUILD_TIME"      # ✅ Có sẵn
```
**Ví dụ**
```
build:
  script:
    - echo "APP_VERSION=1.2.3" > build.env
  artifacts:
    reports:
      dotenv: build.env     # Job sau nhận $APP_VERSION

deploy:
  needs: ["build"]
  script:
    - echo "Version: $APP_VERSION"  # Có sẵn!
```

Quy tắc quan trọng

1. File phải format: KEY=value (không quotes)
2. Chỉ 1 dotenv file per job
3. Phải dùng `needs: ["job-producer"]` 


##### Ví dụ dùng 1 dynamic file cho toàn bộ workflow
```
variables:
  ENV_FILE: "pipeline.env"

generate-config:
  script:
    - apk add jq  # Alpine example
    - |
      cat > $ENV_FILE << EOF
      APP_VERSION=$(cat package.json | jq -r .version)
      GIT_COMMIT=$CI_COMMIT_SHORT_SHA
      BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
      EOF
  artifacts:
    reports:
      dotenv: $ENV_FILE  # Dynamic filename OK!

docker-build:
  needs: ["generate-config"]
  script:
    - docker build --build-arg VERSION=$APP_VERSION .
```

### 3. Cách persist biến giữa các scriptt
1. File-based
```
script:
  - export MY_VAR="hello"
  - echo "$MY_VAR" > /tmp/myvar.env
  
after_script:
  - export MY_VAR=$(cat /tmp/myvar.env)
  - echo "After: $MY_VAR"  # ✅ OK: hello
```
2. YAML variables (Static values) - Định nghĩa ở đầu job
```
variables:
  MY_VAR: "hello"           # Có sẵn ở tất cả: before/script/after

after_script:
  - echo "After: $MY_VAR"   # ✅ OK
```
3. dotenv artifacts (cách này persist giữa các job luôn - tham khảo mục 2)
```
script:
  - echo "MY_VAR=hello" > vars.env
artifacts:
  reports:
    dotenv: vars.env
```

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
---

Giải thích về tag và image 

"Tag" trong GitLab CI dùng để chỉ định runner nào sẽ xử lý job cụ thể. Ví dụ: nếu bạn đặt tags: [docker], job sẽ chỉ chạy trên runner nào có tag "docker".
​
"Image" là hình ảnh Docker mà job sẽ chạy trên đó. Trong file .gitlab-ci.yml, bạn khai báo image: ubuntu:20.04 để chỉ định môi trường chạy job. Đây là nơi chứa các công cụ, thư viện cần thiết để thực thi các bước trong pipeline.
​

Ví dụ:

Nếu bạn chỉ định tag: linux và image: nginx trong GitLab CI, thì:
- tag: linux nghĩa là job sẽ chỉ chạy trên các GitLab runner nào được gắn tag là "linux" (tức là runner đó được cấu hình để xử lý các job cần môi trường Linux).
- image: nginx nghĩa là job sẽ chạy trong một container Docker sử dụng image nginx làm môi trường thực thi, tức là các bước trong job sẽ được thực hiện bên trong một container nginx


**nếu chỉ định tag nhưng không chỉ định image trong GitLab CI, thì job sẽ được chạy trên host (máy chủ cài runner) chứ không phải trong container Docker, nhưng điều này chỉ xảy ra nếu runner được cấu hình dùng executor là shell.**
​
- Nếu runner dùng executor là docker, thì job luôn chạy trong container, và nếu không chỉ định image, runner sẽ dùng image mặc định được cấu hình trong file config.toml hoặc image mặc định của GitLab (thường là một image cơ bản như alpine).
​- Nếu runner dùng executor là shell, job sẽ chạy trực tiếp trên host mà không qua container, và việc chỉ định tag chỉ giúp chọn runner phù hợp. Nếu runner dùng executor là shell mà bạn chỉ định image trong file .gitlab-ci.yml, thì runner sẽ bỏ qua phần image và job vẫn sẽ chạy trực tiếp trên host (máy chủ), không chạy trong container Docker.

Nếu bạn viết GitLab CI mà không chỉ định tags trong job, thì job đó sẽ có thể được chạy bởi bất kỳ runner nào có sẵn và không bị giới hạn bởi tag cụ thể nào​

- Có thể chỉ định tags ở default section để tất cả các job dùng chung 1 tag
---

Khối default trong GitLab CI 

Dùng để áp dụng mặc định cho toàn bộ pipeline. Các tùy chọn phổ biến bao gồm:

- image: Chỉ định Docker image mặc định cho các job.
​- tags: Chỉ định runner tags mặc định.
- before_script: Các lệnh chạy trước mỗi job.
- after_script: Các lệnh chạy sau mỗi job.
- cache: Cấu hình cache mặc định.
- artifacts: Cấu hình artifact mặc định.
- retry: Cấu hình số lần retry khi job thất bại.
- timeout: Thời gian timeout mặc định cho job.
- interruptible: Cho phép hủy job khi có pipeline mới.​

Ví dụ:
```
default:
  image: alpine:latest
  tags:
    - docker
  before_script:
    - echo "Setting up environment"
  after_script:
    - echo "Cleaning up"
  cache:
    paths:
      - vendor/
  artifacts:
    paths:
      - build/
  retry: 2
  timeout: 10 minutes
  interruptible: true
```

---

Executor trong GitLab CI: là cơ chế thực thi job được định nghĩa trong file config.toml của GitLab Runner. Nó quyết định môi trường nào sẽ chạy các lệnh script trong .gitlab-ci.yml.

Executor giống như "driver" cho Runner: nhận job từ GitLab → chuẩn bị môi trường → chạy script → trả kết quả.

Các loại Executor phổ biến
1. Shell Executor
```
[ GitLab Runner ] → Chạy script trực tiếp trên host machine
```

Ưu điểm: Nhanh, đơn giản, tận dụng đầy đủ tài nguyên host.

Nhược điểm: Không cô lập (job A ảnh hưởng job B), khó scale.

Dùng khi: Deploy trực tiếp lên server (như server:deploy nếu dùng shell).

2. Docker Executor (phổ biến nhất)
```
[ GitLab Runner ] → Tạo container Docker → Chạy job bên trong
```

Mỗi job chạy trong container riêng với image chỉ định (image: docker:latest).

Ưu điểm: Cô lập tốt, dễ setup môi trường (AWS CLI, Docker sẵn trong image).

Nhược điểm: Cần Docker daemon trên host runner.

3. Kubernetes Executor
```
[ GitLab Runner ] → Tạo Pod Kubernetes → Chạy job
```

Ưu điểm: Scale tự động theo cluster K8s.

Phù hợp môi trường cloud-native như EKS (AWS).

---

Để GitLab pipeline chỉ trigger khi bạn push lên nhánh master, bạn có hai cách chính phổ biến trong file .gitlab-ci.yml:

1. Dùng workflow: rules (áp dụng cho toàn bộ pipeline)
Đặt ở cấp “root” của file, không nằm trong job:
```
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == "master"
```

Cách này nghĩa là: pipeline chỉ chạy nếu commit được push vào nhánh master; các nhánh khác sẽ không trigger pipeline.

2. Dùng rules cho từng job (nếu muốn giữ một số job không giới hạn)
Nếu không muốn giới hạn toàn bộ pipeline, chỉ muốn một số job chạy trên master:
```
deploy-job:
  stage: deploy
  script:
    - echo "Deploy to production..."
  rules:
    - if: $CI_COMMIT_BRANCH == "master"
```

Trong ví dụ trên, deploy-job chỉ chạy khi push lên master

Lưu ý nhỏ: Trên GitLab mới, nên dùng rules thay vì `only: [master]` vì only/except đã bị deprecated.

---

EOF và 'EOF' khác gì nhau?
EOF và 'EOF' khác nhau ở cách shell xử lý biến và lệnh bên trong heredoc:

1. << EOF (không có dấu ngoặc)
bash
cat << EOF
  Hello $USER
  PWD: $PWD
EOF
Shell sẽ thực hiện biến thay thế và command substitution ($USER, $(pwd), v.v.) trước khi gửi nội dung vào command (ví dụ cat).

Tức là nội dung thực sự gửi đến cat đã là:

text
Hello root
PWD: /home/root
Theo tài liệu Bash, khi delimiter không được quote, shell sẽ thực hiện parameter expansion, command substitution, arithmetic expansion đối với nội dung heredoc.

2. << 'EOF' (hoặc << "EOF")
bash
cat << 'EOF'
  Hello $USER
  PWD: $PWD
EOF
Shell không thay thế biến/command bên trong heredoc, nội dung được giữ nguyên dạng literal.

cat nhận đúng là:

text
Hello $USER
PWD: $PWD
Khi delimiter có bất kỳ dấu ngoặc nào (đơn hoặc kép), nội dung heredoc được xử lý như string literal, tức là không có biến hay command substitution.

3. Áp dụng vào GitLab CI
Nếu dùng << EOF (không quote) trong .gitlab-ci.yml, GitLab CI sẽ thay thế biến ${APP_PATH}, nên remote server nhận cd "/var/www/html/buzz-ticket" → đúng.

Nếu dùng << 'EOF', cd "${APP_PATH}" sẽ được gửi nguyên xi, mà nếu remote server không có APP_PATH thì lệnh sẽ sai hoặc thất bại.

👉 Tóm lại:

EOF → expand biến (GitLab CI / shell local thay thế trước).

'EOF' → không expand, giữ nội dung như string literal.
