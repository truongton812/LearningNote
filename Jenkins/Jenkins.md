###

gitlab webhook: có thể chọn event để trigger đến từ nhánh nào. Khi đó chỉ khi có sự thay đổi trên nhánh đấy thì gitlab mới gửi hook

jenkins pipeline configuration > SCM > Branches to build: chỉ định nhánh cho Jenkins theo dõi. Nếu chỉ chỉ định 1 specific nhánh thì jenkins sẽ chỉ quét và run trên nhánh đó mỗi khi có webhook (không quan tâm gitlabci gửi webhook đến từ nhánh nào)

VD: gitlab webhook cấu hình là push event lên nhánh develop, jenkins pipeline để Branches to build là nhánh main

-> Khi push code lên nhánh develope, jenkins sẽ kéo code ở nhánh main về để run pipeline

ChatGPT:
GitLab webhook có thể cấu hình để chỉ trigger khi có sự kiện (event) xảy ra trên một nhánh cụ thể (ví dụ, chỉ push trên nhánh develop) là đúng.

Khi có thay đổi trên nhánh được chọn trong webhook GitLab, GitLab sẽ gửi webhook đó đến Jenkins hoặc hệ thống khác.

Trong Jenkins pipeline configuration > SCM > Branches to build, bạn chỉ định nhánh mà Jenkins sẽ quét và chạy pipeline.

Nếu trong Jenkins chỉ định một nhánh cụ thể (ví dụ nhánh main), thì Jenkins sẽ luôn lấy code từ nhánh đó để chạy pipeline mỗi khi nhận webhook, bất kể webhook đó gửi từ nhánh nào.

Do đó, trường hợp:

GitLab webhook cấu hình push event chỉ từ nhánh develop,

Jenkins pipeline cấu hình Branches to build là nhánh main,

Khi push code lên nhánh develop, GitLab gửi webhook, Jenkins nhận webhook nhưng sẽ lấy code từ nhánh main để chạy pipeline.

Điều này có nghĩa là dù webhook đến từ nhánh develop, Jenkins sẽ không chạy code trên develop mà lấy code trên main để chạy.

---

Tuy nhiên:
Nếu trong Jenkins pipeline configuration > SCM > Branches to build bạn chỉ định là "any" (hoặc thường dùng là ký tự wildcard như **), thì Jenkins sẽ không tự động lấy code từ tất cả các nhánh để build mỗi khi nhận webhook.

Thay vào đó, hành vi sẽ như sau:

Jenkins sẽ phù hợp (match) với nhánh được gửi đến trong webhook hoặc nhánh được kích hoạt pipeline thay vì cố gắng build tất cả các nhánh cùng lúc.

Nếu webhook từ GitLab gửi kèm tên nhánh cụ thể (ví dụ nhánh develop), Jenkins sẽ nhận biết và build trên nhánh đó.

Jenkins sẽ chỉ build nhánh mà webhook hoặc trigger chỉ định, không phải tất cả các nhánh cùng lúc.

Việc dùng "any" hoặc wildcard là để Jenkins linh hoạt theo nhánh được trigger mà không cần cấu hình cố định một nhánh cụ thể.

Tóm lại: Với cấu hình "any" hoặc wildcard branch, Jenkins sẽ build chính xác nhánh được trigger trong webhook hoặc trigger event, không phải lấy về và build tất cả các nhánh cùng lúc. Đây là cách làm phổ biến để Jenkins pipeline có thể xử lý nhiều nhánh động trong một cấu hình duy nhất.

---

Cấu trúc giao diện Credentials của Jenkins

<img width="1637" height="568" alt="image" src="https://github.com/user-attachments/assets/0cf7b7de-8dce-4b4b-8256-e5a9a2a6d80f" />

- Credential: là tất cả credential trong store `System`:
   - Store: Tham chiếu đến `Store scoped to Jenkins` ở dưới
   - Domain: có 2 loại là global dùng chung cho mọi job và domain riêng (VD mag, IT project) để giới hạn credential cho từng nhóm job.
​   - ID: giá trị để tham chiếu trong Jenkinsfile (ví dụ gitlab-jenkins-token).

Như trong hình, credential `gitlab-jenkins-token` nằm ở `Store: System`, `Domain: IT project`

- Stores scoped to Jenkins: chỗ lưu lớn nhất, có thể là System và đôi khi có User;
  - Trong System luôn có domain mặc định là `Global credentials (unrestricted)` - là nơi lưu trữ credential dùng chung nhiều job. Khi bấm vào `Global credentials (unrestricted)` rồi Add Credentials, credential sẽ thuộc domain global
  - Có thể tạo các domain khác như `mag`, `IT project`. Nếu bấm vào domain `IT project` rồi add thì credential chỉ visible cho các job được cấu hình là dùng domain IT project.

- Cách tạo mới credential để dùng được mọi nơi : Vào: Manage Jenkins → Manage Credentials → System → (global) → Add Credentials. (Nên Chọn Scope = Global trừ khi bạn có lý do bảo mật cụ thể)​
- Vì sao đôi khi không thấy credential trong job để chọn
   - Không đúng domain: Job dùng domain này không thấy credential trong domain khác.
​   - Không đúng scopre: credential được tạo với Scope = System chứ không phải Global, nên chỉ Jenkins core dùng được, job không thấy. Lưu ý muốn xem Scope của credential phải mở chi tiết credential lên
​   - Plugin/SCM bạn đang cấu hình chỉ accept một số loại credential (Kind) nhất định; nếu Kind không khớp (ví dụ tạo Secret text nhưng chỗ đó đang filter Username with password) thì dropdown sẽ ẩn credential đó.
---

### Get the Output of a Shell Command in a Jenkins Pipeline

1.  Using the sh returnstdout
   
The sh step command is a built-in Jenkins pipeline step that allows developers to execute a shell command and capture its output into a variable. Using shell commands, we can assign variables to strings. In a Jenkins pipeline, we can use the sh step to execute a shell command within a pipeline. Let’s look at a pipeline job to execute a command and store its output:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                    def output = sh(returnStdout: true, script: 'pwd')
                    echo "Output: ${output}"
                }
            }
        }
    }
}
```
The pipeline job above defines a stage `Example` along with its steps to execute, including the script element. This special step enables the execution of arbitrary Groovy code within the pipeline. Also, it is utilized to define a Groovy closure that captures shell output. Here, we performed the pwd command inside the script element of the pipeline using the sh returnStdout method. Also, we stored the result in the output variable. The option `“returnStdout: true”` instructs Jenkins to return the standard output of the shell command. The option `“script: ‘pwd'”` specifies the shell command to execute.
Finally, the echo command prints the value of the “output” variable to the Jenkins console using the echo step.  

2.  Using Command Substitution
   
Command substitution is a shell feature that allows developers to execute a command within a string and use its output as part of the string. In the Jenkins pipeline, we can use command substitution to capture the output of a shell command into a variable:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                    def output = sh(script: "echo \$(ls)", returnStdout: true)
                    echo "Output: ${output}"
                }
            }
        }
    }
}
```
In this case, the sh step executes the command `“echo $(ls)”` and captures its output into a variable using command substitution. Furthermore, the pipeline job captures the output of the `“ls”` command into the `“output”` variable and prints it to the Jenkins console upon execution.

3. Other Possible Cases  
Jenkins pipeline provides two other important options which are very useful in storing the result to further use in the pipeline job. The returnStatus method captures the exit status of a shell command, whereas returnStdoutTrim is helpful in trimming the output for further use.

- 3.1.  Using the returnStatus  

The Jenkins pipeline’s sh step uses the returnStatus option to capture the exit status of a shell command. By default, the sh step returns the standard output of the shell command as the result of the step. However, when returnStatus is set to true, the exit status of the shell command is returned instead of the standard output.
Let’s look at the pipeline job with returnStatus method:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                    def status = sh(returnStatus: true, script: 'ls /test')
                    if (status != 0) {
                        echo "Error: Command exited with status ${status}"
                    } else {
                        echo "Command executed successfully"
                    }
                }
            }
        }
    }
}
```
In this example, the sh step executes the ls command and captures the output into the status variable using returnStatus. Since the directory /test doesn’t exist, the ls command will exit with a non-zero status, indicating an error. The if statement checks if the status variable is non-zero, and if so, prints an error message to the Jenkins console.
Developers can perform error checking by capturing the exit status of shell commands. Furthermore, they can also implement logic based on the success or failure of a command.  

- 3.2.  Using the Trim Method

In the following snippet The returnStdout parameter is set to true, which means that the standard output of the command will be captured and returned as the result of the step. Additionally, we can also use the trim() method to get the cleansed output of the command in a variable:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                  def output = sh(returnStdout: true, script: 'echo "    hello    "').trim()
                  echo "Output: '${output}'"
                }
            }
        }
    }
}
```
The trim() method will remove any leading and trailing whitespace characters from the output of the echo command. Furthermore, the Jenkins console will display the resulting output enclosed in single quotes to highlight any whitespace characters.

---
### Sample
##### **Sample 1**
```
#!/usr/bin/env groovy
pipeline {
    agent any #agent tag, which specify which agent would run this pipeline
    environment {
        demo = "SECRET"
    }
    stages {
        stage('build') {
             agent {
   Label ‘aws-amazon’ #specify agent with label ‘aws-amazon’ would run this stage
}
            when {
                expression { BRANCH_NAME ==~ /(test|staging)/ }
            }
            steps {
                docker_loggin("docker_repository","docker.medpro.com.vn")
                docker_helper("docker.medpro.com.vn","thanhtruc" , "client")
                docker_helper("docker.medpro.com.vn","thanhtruc" , "server")
            }
        }
//        stage('test'){
//            steps{
//                test()
//           }
//        }
        stage('deploy'){
            when {
                expression { BRANCH_NAME ==~ /(test|staging)/ }
            }
            steps {
                docker_deploy()
            }
        }
    }
}

//helper

/// Docker help to deploy the image to docker hub .
def docker_helper(repository ,name , folder ) {
    sh """docker build -t $repository/$name -f $folder/Dockerfile ./$folder"""
}
def docker_loggin(credentials, url ) {
    withCredentials([usernamePassword(credentialsId: """$credentials""" ,usernameVariable: """repository_username""", passwordVariable: """repository_password""")]){
        sh """docker login $url -u $repository_username -p $repository_password """
    }
### take the credential from jenkins based on credential id, then extract username and password by built-in variable usernameVariable and passwordVariable and bind to 2 variables repository_username and repository_password
}
def docker_deploy() {
    sh 'docker-compose down '
    sh 'docker-compose up -d --no-color --wait --build --force-recreate'
    sh 'docker-compose ps '
}

def test() {
    def result = sh(script:"""curl -I http://localhost:8000 2>/dev/null | head -n 1 | awk '{print "${2}"}'""",returnStdout: true).trim()
    echo '$result'
    if (result != 200 ) {
        currentBuild.result = 'FAILURE'
    } else {
        currentBuild.result = 'SUCCESS'
    }
}
```


##### **Sample 2**
```
#!/usr/bin/env groovy
pipeline {
    agent {
        label "linux1"
    }
    // environment {
    //     TERRAFORM = tool name: 'terraform', type:  'com.cloudbees.jenkins.plugins.customtools.CustomTool'
    // }
    tools {
          terraform "terraform"
          gradle "gradle"
    }

    parameters {
        choice(
      name: 'ENV',
      choices: [" " ,'DEV', 'PROD'],
      description: 'Passing the Environment'
    )
    }
    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {

        stage('build') {
            when {
                allOf {
                    buildingTag()
                }
            }
            steps {
                script {
                def tagImage = getTagVersion()
                docker_loggin("docker_repository","docker.medpro.com.vn")
                docker_helper("docker.medpro.com.vn","client" , "client",tagImage)
                docker_helper("docker.medpro.com.vn","server" , "server",tagImage)
                }
            }
        }
        stage("Lint and Build helm Chart with Tag "){
            when {
                allOf {
                    buildingTag()
                }
            }
            steps {
                script {
                def tagHelmChart = getTagVersion()
                sh "gradle helmPackage -PChartVersion=$tagHelmChart"
                withCredentials([usernamePassword(credentialsId: 'gradleconfig' ,usernameVariable: 'username', passwordVariable: 'password')]){
                    sh "gradle  helmPublish -Pusername=$username -Ppassword=$password  -PChartVersion=$tagHelmChart"
                }
            }
        }
        }
        stage('deploy'){
            when {
                allOf {
                    buildingTag()
                }
            }
            steps {
                script{
                    dir("k8s"){
                        k8s_deploy()
                    }
                }
            }

        }
    }
}

//helper

/// Docker help to deploy the image to docker hub .
def docker_helper(repository ,name , folder ,tagImage) {
    sh "docker build -t $repository/$name:$tagImage -f $folder/Dockerfile ./$folder"
    sh "docker push $repository/$name:$tagImage"
}
def docker_loggin(credentials, url ) {
    withCredentials([usernamePassword(credentialsId: """$credentials""" ,usernameVariable: """repository_username""", passwordVariable: """repository_password""")]){
        sh """docker login $url -u $repository_username -p $repository_password """
    }
}
// if branch = dev config = devconfig , else kubeproduction

def k8s_deploy() {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh 'mkdir -p ~/.kube'
        sh 'cp $KUBECONFIG ~/.kube/config'
        sh "kubectl apply -f ."
}
}
def test() {
    def result = sh(script:"""curl -I http://localhost:8000 2>/dev/null | head -n 1 | awk '{print "${2}"}'""",returnStdout: true).trim()
    echo '$result'
    if (result != 200 ) {
        currentBuild.result = 'FAILURE'
    } else {
        currentBuild.result = 'SUCCESS'
    }
}
def getTagVersion(){
    def tagImage = params.ENV == ''  ? "${env.GIT_COMMIT}" : sh(returnStdout:  true, script: "git tag --contains").trim()
    return tagImage
}
```


##### **Code to get hash commit**
```
pipeline {
    agent any
    stages {
        stage("Get hash") {
            steps {
                dir("/var/jenkins_home/workspace/My first pipeline/aws-tinhocthatladongian-webserver") {
                    script {
                        CURRENT_HASH = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    }
                    echo "${CURRENT_HASH}"
                }
            }
        }
    }
}
```
##### **Other way (define in function)**
```
pipeline {
    agent any
    stages {
        stage("Get hash") {
            steps {
                dir("/var/jenkins_home/workspace/My first pipeline/aws-tinhocthatladongian-webserver") {
                    script{
                        def CURRENT_HASH = getHash()
                    }
                    echo "${CURRENT_HASH}"
                }
            }
        }
    }
}

def getHash(){
    script{
        CURRENT_HASH = sh(script:'git rev-parse HEAD', returnStdout: true)
    }
    return CURRENT_HASH
}
```

[Configure Build Tools in Jenkins and Jenkinsfile](https://www.youtube.com/watch?v=L9Ite-1pEU8&t=73s)

[Custom Tools Plugin](https://plugins.jenkins.io/custom-tools-plugin/)


https://docs.gitlab.com/integration/jenkins/
