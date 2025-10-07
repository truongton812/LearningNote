# CloudFormation

## I. Định nghĩa
- Là dịch vụ giúp tạo và quản lý infra bằng code (IaC).
- Các khái niệm chính:
  - Template: là code để hướng dẫn CloudFormation tạo infra. Muốn update hạ tầng thì ta upload new version của template (không edit template được).
  - Stack: là 1 instance của template, chứa tất cả các resources được define trong template. Khi delete stack sẽ delete tất cả resources của stack đó.
  - Change set: là list những thay đổi khi update template ⭢ tạo change set để review trước khi thực thi.
- Work flow hoạt động:

```
         upload        refer                create        create
Template ──────▶ S3 ◀────── CloudFormation ──────▶ Stack ──────▶ resources
```
- Trong 1 template có các components:
  - AWS Template Format Version (optional)
  - Description (optional)
  - Resources (mandatory)
  - Parameters: dynamic input cho template (VD lấy từ SSM Parameter Store, Secret Manager,…)
  - Output: reference những resources đã tạo
  - Conditionals: list các điều kiện, rẽ nhánh

##  II. CloudFormation template’s components:

### 1. Resources
- Là thành phần bắt buộc trong template
- Resource type có dạng: service-provider::servicename::data-type-name. VD: AWS::EC2::Instance

### 2. Parameters
- Là pre-input để đưa vào template (Lúc tạo stack sẽ trigger để user khai báo)
- Parameter có các options sau:
  - Type:
    - String
    - Number
    - CommaDelimitedList
    - List<number>
    - AWS-specific Parameter: để catch invalid value hoặc match với existing values trong AWS account, VD lấy key name hoặc VPC ID
    - List <AWS-specific Parameter>. VD: “List<AWS::EC2::Subnet::Id>”
    - SSM parameter: lấy parameter từ SSM Parameter Store
  - Description: mô tả cho parameter
  - Constraint description: mô tả khi constraint không thỏa mãn. VD không đúng allowed value, không đúng min/max value, sẽ hiện warning, ta define trong Constraint Description.
  - Min/MaxLength
  - Min/MaxValue
  - Default
  - AllowedValues: là dropdown để chọn các giá trị
  - AllowedPattern: allow theo dạng regex. VD: regex của IP là `(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})`
  - NoEcho: hiển thị dưới dạng *** trên console và trong log.
- Parameter không được dùng ở AWSTemplateFormatVersion, Description, Transform và Mapping

#### Ví dụ sử dụng Parameter

##### a. Tạo EC2
```
Parameters:
  InstanceType:    #parameter name, có thể đặt tùy ý
    Type: String
    AllowedValues:
    - t2.micro
    - t2.small
    Default: t2.micro
    ConstraintDescription: must be a valid EC2 type
  SecurityGroupPort:     #parameter name
    Description: port allowed
    Type: Number
    MinValue: 1150
    MaxValue: 65535
  KeyName:     #parameter name
    Description: Name of an existing EC2 KeyPair to enable SSH access to the instances
    Type: AWS::EC2::KeyPair::KeyName # AWS-specific parameter, Khi khai báo parameter này trong template CloudFormation, người dùng sẽ thấy một dropdown list các key pair đã có sẵn trong tài khoản để chọn. Việc này cho phép người dùng chọn key pair để sử dụng lúc tạo EC2 instance mà không cần hardcode tên key pair trong template.
  VPCId:
    Type: AWS::EC2::VPC::Id # AWS-specific parameter, giúp list ra các VPC có trong account cho người dùng chọn
Resources:
  MyInstance:     #resource name
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType    #tham chiếu đến parameter bằng hàm !Ref
```
##### b. Lấy parameter từ SSM Parameter Store

```
Parameters:
  Parameter1:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /dev/ec2/InstanceType # mặc định sẽ lấy value của SSM parameter với key là /dev/ec2/InstanceType
  Parameter2:
    Type: AWS::SSM::Parameter::ValueAWS::EC2::Image::Id
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2   #Lấy ra được AMI ID, đây là key có sẵn của AWS trong parameter store (tìm trong public parameter “ami-amazon-linux-latest”)
```

Lưu ý:
- CloudFormation sẽ resolve từ key ra value lưu trong parameter store
- CloudFormation luôn lấy latest value (không thể chọn version)
- CF sẽ không lưu secure string
- Support các type:
  - AWS::SSM::Parameter::Name
  - AWS::SSM::Parameter::Value<String>
  - AWS::SSM::Parameter::Value<CommaDelimitedList> hoặc AWS::SSM::Parameter::Value<List<String>>
  - AWS::SSM::Parameter::Value<AWS-specific-parameter>
  - AWS::SSM::Parameter::Value<List<AWS-specific-parameter>>

### 3. Pseudo Parameters
- Là built-in parameter của AWS, bao gồm 1 số Pseudo Parameters quan trọng:
  - AWS::AccountID ⭢  ID của AWS account mình
  - AWS::Region
  - AWS::StackID
  - AWS::StackName
  - AWS::NotificationARNs
  - AWS::NoValue ⭢  không return value

### 4. Mapping
- Là static var trong template
- Dùng khi có nhiều env (dev, prod,...), nhiều region, nhiều AMI type...

Ví dụ:
```
Mappings:
  RegionMap:    #mapping name
    us-east-1:
      HVM64: <ami-id-1>
      HVMG2: <ami-id-2>
    us-west-1:
      HVM64: <ami-id-1>
      HVMG3: <ami-id-2>
#Sử dụng mapping vào trong resource
Resources:
  MyEC2Instance:    #resource name
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
```


### 5. Output
- Dùng để lấy thông tin của resource trong 1 stack, output này có thể dùng cho các stack khác (cần phải export ).
- Output có thể custom chứ không nhất thiết phải là built-in output của AWS.

Ví dụ output security group

```
Outputs:
  SSHSecurityGroup:     #output name
    Value: !Ref <MySecurityGroup>   #get được Security Group name/ID
    Export:    #Export để cho các stack khác có thể dùng để Security Group tên SSHSecurityGroup (lưu ý tên phải unique across region). Nếu không export mà chỉ dùng output thì chỉ in thông tin ra màn hình
      Name: SSHSecurityGroup    
```
Import thông tin vào stack khác:

```
Resources:
  MyInstance:
    Properties:
      SecurityGroups:
        !ImportValue SSHSecurityGroup
```
Lưu ý: Nếu stack 2 đang ref đến stack 1, không thể delete stack 1.

### 6. Condition
- Dùng để control việc tạo resource hoặc output. VD: khi parameter env = dev ⭢ chỉ tạo EC2, còn khi env = prod ⭢ tạo EC2, ELB,...
- Condition có thể refer đến condition khác, parameter value hoặc mapping

Ví dụ
```
Conditions:
  CreateProdResources: !Equals [!Ref EnvType, prod]
  (name, tùy ý)         (ngoài Equals còn có And/Or/If/Not)
Resources:
  MountPoint:
    Type: AWS::EC2::VolumeAttachment
    Condition: CreateProdResources    #Nếu condition = true thì tạo mountpoint
```
##  III. Intrinsic function
- Là các hàm tích hợp sẵn trong CloudFormation cho phép thực hiện các phép toán và thao tác trong template của CloudFormation
- Một số function thông dụng: Ref, GetAtt, FindInMap, ImportValue, Condition, Base64,...

### 1. Hàm Ref
- Dùng để lấy giá trị của 1 resource hoặc parameter
- VD: Ref của AWS::EC2::Instance là instance ID
- Do Ref lấy được giá trị của cả resource và parameter ⭢ logical ID của resource và parameter phải khác nhau

### 2. Hàm GetAtt
- Dùng để lấy giá trị của thuộc tính cụ thể của một resource sau khi khởi tạo
- VD
```
Resources:
  EBSVolume:
    Type: AWS::EC2::Volume
    Properties:
      Size: 100
      AvailabilityZone: !GetAtt MyInstance.AvailabilityZone
```

### 3. Hàm Base64
- Dùng để convert string thành base64, thường dùng để đưa data vào EC2 User data
- User data log được lưu tại /var/log/cloud-init-output.log.

## IV. CloudFormation Rollback
- Khi tạo stack mà fails có 2 options (config trong “stack failure option”):
  - Rollback (default): xóa tất cả resources.
  - Disable rollback: giữ lại các resources đã tạo thành công ⭢ có thể troubleshoot, còn các resources fail sẽ bị rollback.
- Khi update stack mà fails có 2 options:
  - Automatically rollback (default): tự động rollback về version trước (version hoạt động bình thường), và delete tất cả resources mới tạo fail.
  - Giữ lại các resources tạo thành công và rollback các resources bị fail.
- Lưu ý:
  - Khi tạo stack mà fails thì chỉ có duy nhất 1 option là delete stack chứ không update lại bằng template mới được
  - Khi stack update fails ⭢ rollback mà rollback tiếp tục fails ⭢ cần phải “fix” resource manually và issue lại ContinueUpdateRollback API.

## V. CloudFormation Service Role
- Là IAM role cho phép CloudFormation tạo/update/xóa resource. Mỗi khi tạo stack ta sẽ gắn service role này vào stack ⭢ stack sẽ dùng role này để vận hành các resource.
- Nếu không gán role cho stack thì mặc định stack sẽ dùng role/permission của user tạo ra stack. Lưu ý là permission này chỉ áp dụng lúc stack thực hiện lần đầu. Ví dụ user A có quyền tạo S3 và dùng CloudFormation để tạo S3 thì sẽ thành công được, tuy nhiên sau đó user B không có quyền tạo S3 mà update stack thì sẽ bị failed.
- Với CloudFormation Service Role, ta gán quyền trực tiếp cho stack ⭢ không cần quyền của user để vận hành stack. Lưu ý user cần có quyền passRole (iam:PassRole là quyền dùng để grant 1 role cụ thể cho user/service khác).
- Ví dụ: Tạo Service role tên CloudFormationS3Create để cho phép tạo S3 bucket
```
#Trust policy:
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "cloudformation.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```
#Permissions policy:
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```
Sau đó ta có thể gán quyền passRole cho user A ⭢ User A có thể tạo CloudFormation stack để tạo S3 bucket mà không cần trực tiếp có quyền tạo S3 bucket.
```
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::<account-id>:role/CloudFormationS3Create"
}
```



## VI. CloudFormation Capabilities
- CAPABILITY_NAMED_IAM và CAPABILITY_IAM: cần phải tick chọn các option này nếu muốn dùng CloudFormation để tạo/update 1 IAM resource (IAM user/role/group/policy/access key/instance profile/...).
  - Specify CAPABILITY_NAMED_IAM nếu resource được đặt tên.
  - Specify CAPABILITY_IAM nếu resource không được đặt tên.
- CAPABILITY_AUTO_EXPAND: dùng để xác nhận CloudFormation template có thể thay đổi trước khi deploy ⭢ cần tick chọn nếu trong template có dùng Macros hoặc Nested Stack.
- Nếu launch template bằng API mà trả về lỗi InsufficientCapabilitiesException thì nguyên nhân là do thiếu xác nhận capability 

## VII. CloudFormation Deletion Policy
- Dùng để control behavior khi stack bị xóa hoặc resource bị remove khỏi stack.
- Định nghĩa deletion policy trong resource block của template.
- Deletion policy default là Delete: khi stack bị xóa ⭢ tất cả resources bị xóa theo. Lưu ý có exception đối với S3: nếu trong S3 bucket còn object thì không thể xóa được, cần phải delete objects thủ công hoặc tạo custom resource (Lambda) để xóa.
- Deletion policy options:
  - Retain: giữ lại resource nếu stack bị xóa. VD:
```
Resources:
  MyDynamoDBTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain
```
  - Snapshot: tạo snapshot của resource sau khi xóa resource, chỉ áp dụng cho 1 số resource:
    - EBS volume, ElastiCache Cluster, ElastiCache Replication Group
    - RDS DB Instance, RDS DB Cluster, Redshift Cluster, Neptune DB Cluster, DocumentDB DB Cluster

## VIII. Stack Policy
- Khi update 1 stack, mặc định CloudFormation sẽ update tất cả resources của stack. Ta có thể dùng stack policy để control resource nào không được update khi update stack.
- Stack policy chỉ assign với stack lúc khởi tạo stack.
- Nếu muốn update 1 resource được protected, cần sửa policy thành allow.
- Ví dụ 1 stack policy không cho phép update VPC resource:

```
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "Update:*",    #Có thể chi tiết hơn bằng cách dùng Update: Modify/Delete/Replace
      "Principal": "*",
      "Resource": "*" 
    },
    {
      "Effect": "Deny",
      "Action": "Update:Replace",
      "Principal": "*",
      "Resource": "LogicalResourceId/VPC",
      "Condition": {
        "StringLike": {
          "ResourceType": ["AWS::EC2::VPC"]
        }
      }
    }
  ]
}
```
## IX. Termination Policy
- Dùng để tránh xóa nhầm stack.
- Enable bằng cách chọn stack ⭢ Stack actions ⭢ Activate ⭢ Termination protection.

## X. Custom resource
- Là resource trong CloudFormation giúp mở rộng khả năng của CloudFormation bằng cách gọi các dịch vụ hoặc logic tuỳ chỉnh trong quá trình triển khai stack. Có thể là:
  - Chạy 1 lambda function và lấy kết quả trong khi đang create/update/delete stack.
  - Tạo resource dựa trên logic.
  - Tạo resource mà CloudFormation chưa support hoặc non-AWS resource (on-prem, third-party).
- Cách hoạt động:
  - CloudFormation tạo custom resource ⭢ CloudFormation gửi event create/update/delete đến 1 specific endpoint.
  - Endpoint xử lý yêu cầu (thường là Lambda).
  - Endpoint trả kết quả cho CloudFormation: gửi HTTP response đến 1 pre-signed S3 URL mà CloudFormation cung cấp.
  - CloudFormation tiếp tục triển khai stack dựa vào kết quả trả về từ Custom Resource.
- Cách tạo custom resource:
```
  Type: Custom::<Name>
  ServiceToken:     #endpoint mà CloudFormation gọi vào để xử lý, có thể là
                      - lambda arn ⭢ gọi lambda xử lý trong quá trình deploy stack.
                      - SNS topic arn ⭢ gửi sự kiện đến SNS để các subscriber (VD: lambda) xử lý.
                      - SQS Queue
```

- Use case:
  - Empty S3 bucket trước khi delete stack có chứa S3 bucket.
  - Gọi API AWS không có trong CloudFormation (VD: tạo tài khoản AWS Organization, bật WAF logging).
- Điểm khác nhau giữa tạo Custom resource và tạo Lambda bằng CloudFormation template:

| | Lambda function	| Custom resource |
|---|---|---|
| Tạo lambda function |	Có	| Không (chỉ gọi lambda có sẵn) |
| Chạy lambda ngay trong deploymen | Không | Có |
| Trả kết quả về CloudFormation | Không | Có | 
| Gọi API không có trong CloudFormation | Không | Có | 

## XI. Dynamic Reference
- Ta có thể refer value trong SSM Parameter Store và Secret Manager vào CloudFormation template ⭢ CloudFormation sẽ retrieve value lúc create/update/delete stack.
- Syntax: {{resolve:<service-name>:<reference-key> }}
- Ví dụ: lấy plain text store trong SSM
```
Resource:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: '{{resolve:ssm:S3AccessControl:2}}'  #trong đó ssm là service name; S3AccessControl là reference key; 2 là version của parameter
```
- Ví dụ: Lấy secure string trong SSM:
```
Password: '{{resolve:ssm-secure:IAMUserPassword:10}}'
```
- Ví dụ lấy secret value trong Secret Manager:
```
Password: '{{resolve:secretsmanager:MyRDSSecret:SecretString:password}}'
```
