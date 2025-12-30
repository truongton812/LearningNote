Cấu trúc 1 file Jenkins 

```
pipeline {                          // Level 1: Root (bắt buộc)
  agent any                         // Level 2: Global agent
  stages {                          // Level 2: Stages container (bắt buộc)
    stage('Name') {                 // Level 3: Individual stage (bắt buộc ≥1)
      steps {                       // Level 4: Steps container (bắt buộc)
        sh 'command'                // Level 5: Actual steps
        script {                    // Level 5: Groovy scripting (optional)
          env.VAR = 'value'
        }
      }
    }
  }
}
```

Trong `steps` là các `step`. Step có thể là
- sh
- script {}
- plugin `with...{}` . VD `withCredentials{}`. Trong dấu ngoặc `{ ... }`: plugin inject tạm thời các biến môi trường (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, …) cho các step bên trong. Sau dấu `}` các biến đó bị xóa khỏi môi trường build, nên các lệnh phía sau không còn quyền / không thấy credential nữa.

VD1:
```
steps {
  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'terraform-user']]) {
    sh 'aws sts get-caller-identity'   // ✅ Ở ĐÂY mới có AWS_ACCESS_KEY_ID,...
  }                                    // ⬅ Hết block → env tạm bị gỡ
  sh 'aws sts get-caller-identity'     // ❌ Ở đây KHÔNG còn credential nữa
}
```

VD2:
```
withCredentials([...]) {
  sh 'aws ec2 create-image ...'       // ✅ dùng được AWS
}
env.NEW_AMI_ID = sh(script: 'aws ec2 describe-images ...', returnStdout: true)  // ❌ bước này ở ngoài scope
```

Lưu ý:
- `env.VAR = ...` chỉ được đặt trong `script {}` do đây là biểu thức Groovy
- Trong `script {}` khi sử dụng biểu thức Groovy hoặc cần lấy giá trị trả về thì không được dùng `sh 'command'` mà phải dùng sh function `sh(script: '...', returnStdout: true)`. VD (tham khảo thêm ở dưới)

Cấu trúc các step trong 1 steps
```
steps {
    sh 'ls'                           // ✅ Step syntax
    script { 
      result = sh(script: 'ls', returnStdout: true)  //  ✅ phải dùng dạng function do đây là biểu thức Groovy
      env.AMI_ID = sh(script: 'aws ec2 create-image ...', returnStdout: true).trim()  // ✅ phải dùng dạng function do đây là biểu thức Groovy
      sh 'deploy-prod'                // Function call shorthand
      sh "echo hello from sh"              // ✅ Hợp lệ do không dùng Groovy
      def out = sh(script: 'date', returnStdout: true).trim()  // ✅ Hợp lệ
      def out = sh "echo hi"  // ❌ không hợp lệ
    }
    script {                // Script là "jailbreak" để chạy Groovy arbitrary code trong declarative pipeline!
      if (env.BRANCH_NAME == 'main') {
        withAWS { sh 'deploy-prod' }
      } else {
        sh 'deploy-staging'
      }
    }
    withAWS(credentials: 'terraform-user') {
      script {                 // ✅ Wrap tất cả
        sh 'aws ec2 create-image ...'
        env.NEW_AMI_ID = sh(...)  // ✅ Groovy OK trong script {}
      }
      sshagent{} // ✅ Có thể đặt 1 plugin khác
    }
}
```


| Trong `script {}` | `sh 'command'` | `sh(script: '...')` |
|-------------------|----------------|---------------------|
| **Loại** | **Step** (declarative syntax) | **Function** (Groovy method) |
| **Chạy trên** | Jenkins agent | Groovy sandbox |
| **Return** | Void (chạy xong) | String/Int (capture output) |
| **Syntax** | `steps { sh 'ls' }` | `result = sh(script: 'ls', returnStdout: true)` |

- Trong plugin có thể dùng mọi step. VD như sh, script {}, withAWS {}, echo,....  hoàn toàn có thể đặt một with khác (kể cả withAWS{}, withCredentials{}, sshagent{}…) bên trong block withAWS{}/withCredentials{} 
- sh và script {} trong Jenkins Declarative Pipeline khác nhau hoàn toàn về mục đích và syntax được phép. Script{} trả về Groovy object

```
steps {
  sh 'aws ec2 create-image ...'           // Shell command
  sh(script: 'date', returnStdout: true)  // Lấy output
  sh 'echo "Hello"'                       // Đơn giản
  env.VAR = 'value'  // ❌ Lỗi! steps {} không nhận Groovy statements
  script {
    env.AMI_ID = sh(script: 'aws ec2 create-image ...', returnStdout: true).trim()
    if (env.BRANCH_NAME == 'main') {
      sh 'deploy-prod'
    }
    sh 'aws ...'  // ❌ Lỗi! script {} không nhận sh trực tiếp, phải dùng sh() function
  }
}
```

---
## 2. Quote trong jenkins
Trong jenkins, Single quotes dùng cho 1 dòng command đơn giản. Còn triple quotes cần khi multi-line script hoặc complex commands.
### 2.1 Single quotes
Ví dụ
- sh 'echo hello'
- sh 'echo $BUILD_NUMBER' ->  in ra build number trên jenkins. Nguyên nhân là do Jenkins tự động inject tất cả environment variables (BUILD_NUMBER, JOB_NAME, GIT_COMMIT...) vào global shell environment trước khi chạy sh step (có thể check bằng sh 'env'). Nếu ta mở một shell local bằng sh step thì sẽ thừa hưởng các Jenkins variable từ global shell. Các thay đổi tới variable đấy chỉ có hiệu lực trong local shell (tức step sh đang mở)
- sh 'aws ec2 describe-instances \
  --filters "Name=tag:env,Values=test" \
  --query "Instances[0].Id"' -> fail

### 2.1 Triple quotes
Có 2 loại là Triple single quotes ('''...''') và triple double quotes ("""..."""), đều dùng để tạo chuỗi nhiều dòng, nhưng khác nhau ở khả năng interpolation (thay thế biến):

#### 2.1.1. Triple single quotes ('''...'''):
- Tạo chuỗi nhiều dòng nhưng không hỗ trợ interpolation. Các biến như `${variable}` sẽ không được thay thế, mà xuất hiện nguyên bản trong chuỗi. Đây là plain java.lang.String.
- Dùng khi muốn giữ nguyên nội dung (shell script, config file).
- Ví dụ
```groovy
def name = 'Jenkins'
def single = '''Hello ${name}'''  // Output: Hello ${name}
```
​
#### 2.1.2. Triple double quotes ("""..."""):
- Tạo chuỗi nhiều dòng và hỗ trợ interpolation. Biến ${variable} sẽ được thay thế bằng giá trị của nó. Đây là groovy.lang.GString nếu có interpolation, hoặc java.lang.String nếu không.
- Dùng khi cần thay thế biến trong chuỗi.
​- Ví dụ
```groovy
def name = 'Jenkins'
def double = """Hello ${name}"""  // Output: Hello Jenkins
```
- Kinh nghiệm là nên dùng """ (triple double quotes) khi viết sh trong Jenkinsfile, để Groovy expand các `environment`, `parameter` được khai báo trước khi gửi cho Bash. Ví dụ:

```
// ✅ ĐÚNG - Groovy expand trước
sh """
  aws ec2 create-image --name "${amiName}"   # Bash nhận giá trị thật từ env amiName
"""

// ❌ SAI - Bash nhận literal string  
sh '''
  aws ec2 create-image --name "${amiName}"   # Fail do Bash nhận giá trị là chuỗi "amiName"
'''
```

---

### 3. Dấu backslash trong Jenkins
- Dấu \ (backslash) trong Jenkins Pipeline dùng để escape ký tự $ của shell, ngăn Groovy xử lý nhầm thành biến của nó, do Groovy luôn interpolate (thay thế) tất cả ${...} bằng giá trị khai báo trong environment hoặc parameter trước khi gửi xuống shell. 
- Để shell nhận giá trị từ chính shell thì cần escape bằng backslash để bảo vệ biến 
- Ví dụ
```
pipeline {
    agent any
    environment {
        JENKINS_VAR = "This var is from jenkins"
    }
    stages {
        stage('backslash example') {
            steps {
                sh """
                    BASH_VAR="this var is from bash"
                    echo $BASH_VAR //❌ lỗi do Jenkins không tìm thấy BASH_VAR trong khai báo environment/parameter
                    echo \$BASH_VAR //✅ lấy ra được "this var is from bash"
                    echo $JENKINS_VAR //✅ lấy ra được "This var is from jenkins"
                """
            }
        }
    }
}​
```

---

Cần nhớ:

`sh` trong Jenkins là một step, vì vậy chỉ dùng được ở những chỗ Jenkins cho phép gọi steps. Ở một số chỗ, bạn phải gọi sh(...) như hàm Groovy chứ không viết kiểu DSL rút gọn.

Trong Jenkins Pipeline có 2 cách viết `sh`, cả hai đều là gọi Jenkins step sh, chỉ khác syntax và nơi dùng:
- Kiểu DSL (thường thấy trong Declarative) dùng khi không cần output: `sh "echo hello"`
- Kiểu function (hàm của Groovy) thường thấy khi dùng trong biểu thức Groovy hoặc cần lấy giá trị trả về returnStdout, returnStatus: `sh(script: 'echo hello', returnStdout: true).trim()`



Những nơi không dùng được dạng DSL, phải dùng dạng function
- Trường hợp 1: bên trong biểu thức Groovy (gán biến, toán tử)
```
script {
  // ❌ Sai: DSL không phải là expression Groovy
  def out = sh "echo hello"    // Jenkins sẽ báo lỗi do Groovy coi sh "..." là một statement DSL, không phải expression để gán.

  // ✅ Đúng: dùng sh(...) như function Groovy
  def out = sh(script: 'echo hello', returnStdout: true).trim() // đây là gọi hàm trả về string, Groovy hiểu được
  echo "OUT = ${out}"
}
```

- Trường hợp 2: trong toán tử điều kiện / vòng lặp. Trong if, for, Groovy bắt buộc cần expression trả về value, nên phải dùng sh(...) dạng function với returnStatus hoặc returnStdout.
```
script {
  // ❌ Sai
  if (sh "test -f /tmp/file") {
    echo "File exists"
  }

  // ✅ Đúng
  if (sh(script: 'test -f /tmp/file', returnStatus: true) == 0) {
    echo "File exists"
  }

  // ❌ Sai
  for (i in sh "ls") { ... }

  // ✅ Đúng
  def files = sh(script: 'ls', returnStdout: true).trim().split('\n')
  for (f in files) { echo f }
}
```


