# ghichep-prometheus

**MỤC LỤC**

[1. Giới thiệu Prometheus](Doc/01.%20overview.md)

[2. Truy vấn PromQL](Doc/02.%20promql.md)




# JENKINS

## 1. Cấu trúc jenkins file

- Jenkins file có thể viết theo kiểu Declarative hoặc Scripted
  - Declarative Pipeline: Sử dụng cú pháp DSL, bắt đầu bằng pipeline {} với các phần như agent, stages, steps, post. Ưu điểm là đơn giản, nhược điểm là kém linh hoạt trong việc thực hiện logic phức tạp hoặc tùy biến sâu, giới hạn trong cấu trúc được định nghĩa sẵn. Có thể chèn Scripted Pipeline bằng directive `script {}` khi cần
  - Scripted Pipeline: Sử dụng Groovy script, bắt đầu với node {}. Ưu điểm là linh hoạt, có thể tùy chỉnh mọi logic, điều kiện, vòng lặp phức tạp, xử lý lỗi nâng cao. Dễ tái sử dụng và modular hóa code nhờ khả năng viết script Groovy tùy biến.
- Với nhu cầu thông thường thì người dùng hay viết bằng Declarative Pipeline
- Cấu trúc cơ bản của một Jenkinsfile theo kiểu Declarative Pipeline
```
pipeline {
    agent {label 'agent-label'}  // Chỉ định môi trường để chạy pipeline (có thể là any hoặc none)
    
    environment {  // Khai báo các biến môi trường (nếu cần)
        VAR_NAME = "value"
        DOCKERHUB_CREDENTIAL=credentials{'dockerhub'}
        NAME = 'JENKINS'
    }

    stages {  // Tập hợp các giai đoạn (stage) trong pipeline
        stage('Stage Name') {
            steps { 
                sh 'echo Hello World'  
            }
        }
    }
    
    post {  // Các hành động được thực hiện sau khi pipeline hoặc stage kết thúc
        always {
            cleanWs()  // dọn dẹp workspace sau khi chạy xong
        }
    }
}
```




Trong Jenkinsfile kiểu Declarative Pipeline, cú pháp và cấu trúc được quy định rõ ràng nhằm giúp pipeline dễ đọc, dễ bảo trì và giảm lỗi. Tuy nhiên, Declarative pipeline cũng cho phép bạn chèn một đoạn Scripted Pipeline vào bên trong bằng cách dùng directive script {}.
script {} là một khối đặc biệt trong Declarative Pipeline, cho phép bạn viết mã Groovy thuần theo kiểu Scripted Pipeline ngay bên trong pipeline.

Khi cần thực hiện các thao tác phức tạp mà Declarative Pipeline không hỗ trợ hoặc rất khó thực hiện do giới hạn cú pháp, bạn có thể sử dụng script {} để viết mã tùy chỉnh.
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'This is declarative step'
                
                script {
                    // Code Groovy kiểu scripted pipeline ở đây
                    def today = new Date()
                    echo "Today's date is ${today}"
                    
                    if (today.day == 1) {
                        echo 'Today is the first day of the month!'
                    }
                }
            }
        }
    }
}
```


Bạn hoàn toàn có thể đặt một đoạn bash script vào trong Jenkins Pipeline và chạy nó bình thường.

Trong Jenkinsfile, để chạy bash script trên môi trường Linux/Unix, bạn dùng step sh. Ví dụ:


```
pipeline {
    agent any
    stages {
        stage('Run Bash Script') {
            steps {
                sh '''
                #!/bin/bash
                echo "Hello from bash script"
                ls -l
                '''
            }
        }
    }
}
```

Từ khóa sh dùng để chạy shell script trên node Jenkins agent.

Bên trong phần script bạn có thể ghi nhiều dòng lệnh bash như trong terminal, bao gồm khai báo #!/bin/bash nếu muốn.

Bạn cũng có thể gọi trực tiếp file script .sh có sẵn trong workspace, ví dụ:


sh './deploy.sh'

