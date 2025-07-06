# Table of contents
1. [Introduction](#introduction)
2. [Some paragraph](#paragraph1)
    1. [Sub paragraph](#subparagraph1)
3. [Another paragraph](#paragraph2)

## This is the introduction <a name="introduction"></a>
Some introduction text, formatted in heading 2 style

## Some paragraph <a name="paragraph1"></a>
The first paragraph text

### Sub paragraph <a name="subparagraph1"></a>
This is a sub paragraph, formatted in heading 3 style

## Get output <a name="paragraph2"></a>
The second paragraph text

---

###
- Chạy jenkins bằng docker

```Docker run -d - -name jenkins -p 8080:8080 -p 50000:50000 -v /Users/tuna/desktop/jenkins:/var/jenkins_home jenkins/jenkins:lts```

- Cài đặt ngrok
  
```
yum install -y wget unzip
wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
unzip ngrok-stable-linux-amd64.zip
```

- Cài docker bằng scrip:
  
```curl https://releases.rancher.com/install-docker/20.10.sh | sh```

- Quản lý user: cài plugin LBAC  
- Parameter: cho phép truyền các tham số người dùng vào thời điểm bắt đầu chạy pipeline  

- Enviroment: các tham số sử dụng khi chạy pipeline
```
environment {
    DOCKERHUB_CREDENTIAL=credentials{'dockerhub'}
    NAME = 'JENKINS'
]
```
---
Envinronment supports a special helper method credentials()
Supported Credentials Type: Secret Text, Secret File,Username and password, SSH with Private Key  
```
pipeline {
    agent any
    stages {
        stage('Example Username/Password') {
            environment {
                SERVICE_CREDS = credentials('my-predefined-username-password')
            }
            steps {
                sh 'echo "Service user is $SERVICE_CREDS_USR"'
                sh 'echo "Service password is $SERVICE_CREDS_PSW"'
                sh 'curl -u $SERVICE_CREDS https://myservice.example.com'
            }
        }
        stage('Example SSH Username with private key') {
            environment {
                SSH_CREDS = credentials('my-predefined-ssh-creds')
            }
            steps {
                sh 'echo "SSH private key is located at $SSH_CREDS"'
                sh 'echo "SSH user is $SSH_CREDS_USR"'
                sh 'echo "SSH passphrase is $SSH_CREDS_PSW"'
            }
        }
    }
}
```

- Variable scope: variable của stage chỉ dùng đc trong stage  
- Conditional trong Jenkin: Cho phép ta điều khiển luồng chạy của pipeline  
VD1: chỉ chạy stage deploy khi branch là develop
```
Stages {
	stage(‘deploy’) {
		When {
			Branch ‘develop’
		}
		Steps { command }
	}
}
```  
VD2: chỉ chạy stage deploy khi thỏa mãn tất cả các điều kiện trong khối allOf (hoặc anyOf là chỉ cần thỏa mãn 1 trong các điều kiện)  
```
Stages {
	stage(‘deploy’) {
		When {
			allOf {
				Branch ‘develop’
				Environment name: “ten_bien”, value: “gia_tri_bien”
			}
		}
		Steps { command }
	}
}
```
- Stages và Parallel: Cho phép ta lựa chọn thứ tự xử lý các task.  
    + Stages: xử lý tuần tự các task  
    + Parallel: xử lý đồng thời các task  

Các khối stages và parallel có thể nằm lồng trong nhau, tuy nhiên không thể tạo 1 khối parallel bên trong 1 khối stage mà khối stage đó đang nằm trong khối parallel khác  
VD1: 
```
Stages {
	stage(‘Read file’) {
		stages {
			stage(‘1’) {
				Steps{command}
			}
			stage(‘2){
				Steps{command}
			}
		}
	}
}
```
VD2:
```
Stages {
	stage(‘Build in’) {
		parallel {
			stage(‘1’) {
				Steps{command}
			}
			stage(‘2){
				Steps{command}
			}
		}
	}
}
```
- Tạo giai đoạn cuối cho mỗi pipeline (post-build actions): Cho phép ta tạo ra 1 bước cuối cùng ở mỗi stage/pipeline/. Thông thường, để dọn dẹp hoặc để gửi mail/notification thông báo là pipeline đã chạy xong. Khối post này có thể nằm trong khối stages, stage, pipeline,...

VD: Luôn hiển thị dòng finish kể cả có lỗi hay không
```
post {
	Always {
		Echo “finish”
	}
}
```
Ngoài always còn có changed, fixed, regression, aborted, failure, success, unstable, unsuccessful, and cleanup.  

`changed`  
Only run the steps in post if the current Pipeline’s run has a different completion status from its previous run.  
`fixed`  
Only run the steps in post if the current Pipeline’s run is successful and the previous run failed or was unstable.  
`regression`  
Only run the steps in post if the current Pipeline’s or status is failure, unstable, or aborted and the previous run was successful.  
`aborted`  
Only run the steps in post if the current Pipeline’s run has an "aborted" status, usually due to the Pipeline being manually aborted. This is typically denoted by gray in the web UI.  
`failure`  
Only run the steps in post if the current Pipeline’s or stage’s run has a "failed" status, typically denoted by red in the web UI.  
`success`  
Only run the steps in post if the current Pipeline’s or stage’s run has a "success" status, typically denoted by blue or green in the web UI.  
`unstable`  
Only run the steps in post if the current Pipeline’s run has an "unstable" status, usually caused by test failures, code violations, etc. This is typically denoted by yellow in the web UI.  
`unsuccessful`  
Only run the steps in post if the current Pipeline’s or stage’s run has not a "success" status. This is typically denoted in the web UI depending on the status previously mentioned (for stages this may fire if the build itself is unstable).  
`cleanup`  
Run the steps in this post condition after every other post condition has been evaluated, regardless of the Pipeline or stage’s status.  


-Tạo timeout cho pipeline: Cho phép ta giới hạn thời gian mà 1 pipeline được phép build. Điều này ngăn tình trạng vì 1 lý do nào đó mà 1 pipeline chạy vô thời hạn, Dẫn đến tốn tài nguyên.
Dashboard -> any branch -> pipeline syntax -> Declarative Directive Generator -> Sample directive: option -> timeout -> generate ra dòng lệnh
VD:
Option {
	Timeout(time: 1, unit: ‘second’)
}



-Checkout SCM
script:
git credentialsId: '<gitlab-credential>', url: ‘<url>' -> checkout nhánh master
git branch: 'dev', credentialsId: '<gitlab-credential>', url: '<url>' -> checkout các nhánh khác

Lưu ý: script này tương tự option Pipeline script from SCM trong config pipeline, do đó chỉ cần dùng 1 trong 2

[Image](https://drive.google.com/file/d/1-bAmYA8Ccr-fFBqGdNTCUfWWy1p_jPfB/view?usp=sharing)

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
