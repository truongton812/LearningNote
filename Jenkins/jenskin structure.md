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
- plugin with...{} . VD withCredentials{}. Trong dấu ngoặc { ... }: plugin inject tạm thời các biến môi trường (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, …) cho các step bên trong. Sau dấu }: các biến đó bị xóa khỏi môi trường build, nên các lệnh phía sau không còn quyền / không thấy credential nữa.

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
- env.VAR = ... chỉ được đặt trong script {}
- Trong script {} chỉ nhận Groovy function calls hoặc Groovy logic, không nhận declarative steps. Do đó không được dùng `sh 'command'` mà phải dùng sh function `sh(script: '...', returnStdout: true)`. VD

Cấu trúc các step trong 1 steps
```
steps {
    sh 'ls'                           // ✅ Step syntax
    script { 
      result = sh(script: 'ls', returnStdout: true)  //  ✅ OK! Function syntax
      env.AMI_ID = sh(script: 'aws ec2 create-image ...', returnStdout: true).trim()  // ✅ OK!
      sh 'deploy-prod'                // Function call shorthand
      sh 'aws ec2 ...'  // ❌ LỖI! script {} không nhận "steps"
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
