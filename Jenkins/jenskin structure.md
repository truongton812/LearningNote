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
- Trong script{} không được dùng sh mà phải dùng sh function (xem thêm đoạn chat ở dưới)
- Trong plugin có thể dùng mọi step. VD như sh, script {}, withAWS {}, echo,....  hoàn toàn có thể đặt một with khác (kể cả withAWS{}, withCredentials{}, sshagent{}…) bên trong block withAWS{}/withCredentials{} 
