# CloudFormation

## Định nghĩa
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

##  CloudFormation template’s components:

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
  - AWS::AccountID -> ID của AWS account mình
  - AWS::Region
  - AWS::StackID
  - AWS::StackName
  - AWS::NotificationARNs
  - AWS::NoValue -> không return value

### 4. Mapping
- Là static var trong template
- Dùng khi có nhiều env (dev, prod,...), nhiều region, nhiều AMI type...

