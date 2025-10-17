Mối liên hệ giữa Subnet và Availability Zone (AZ) trong AWS như sau:

Một Availability Zone (AZ) là một vị trí vật lý riêng biệt trong một AWS Region. Mỗi AZ gồm một hoặc nhiều trung tâm dữ liệu (data center) độc lập, giúp tăng tính sẵn sàng và khả năng chịu lỗi cho hệ thống.

Một Subnet là một mạng con (subnetwork) được tạo trong một VPC và nằm hoàn toàn trong một Availability Zone duy nhất. Nghĩa là, mỗi subnet chỉ gói gọn trong một AZ, không thể kéo dài qua nhiều AZ.

Khi tạo subnet trong AWS, bạn cần chỉ định CIDR block cho subnet đó và chọn một AZ cụ thể để subnet thuộc về.

Mỗi VPC có thể có nhiều subnet, mỗi subnet nằm trong một AZ khác nhau nhằm phân bố tài nguyên, hệ thống của bạn có thể được triển khai trên nhiều AZ khác nhau để tăng độ dự phòng và khả năng chịu lỗi.

Việc phân chia subnet theo AZ giúp AWS và bạn kiểm soát mạng tốt hơn, phân tách tài nguyên theo vùng vật lý, dễ dàng tổ chức kiến trúc dịch vụ phân tán, đảm bảo khi AZ này gặp sự cố thì các subnet (và tài nguyên) ở AZ khác vẫn hoạt động bình thường.

Khi tạo tài nguyên EC2 trong AWS, bạn cần chỉ định Subnet chứ không phải chỉ định trực tiếp Availability Zone (AZ).

Tóm lại:
Subnet là mạng con được chứa trong một AZ cố định, còn AZ là khu vực vật lý/phần cứng của AWS trong một Region. Mối quan hệ này giúp phân tách và cách ly tài nguyên trong mạng một cách vật lý và logic hiệu quả.



Internet GW là bound với VPC

security group bound với vpc

elb bound với vpc, có thể span trên nhiều az

---

<img width="763" height="355" alt="image" src="https://github.com/user-attachments/assets/ba2a22aa-a743-4c19-aa36-6eef22f9a96f" />


Thiết kế VPC trên hình của bạn là hoàn toàn chuẩn và phù hợp theo các best practice của AWS hiện nay. Cụ thể, bạn đã chia mỗi Availability Zone (AZ1, AZ2, AZ3) thành ba loại subnet riêng biệt: public subnet, private subnet, và db subnet.

Ưu điểm thiết kế này
Đáp ứng chuẩn 3-tier (Web-Application-Database) cho mô hình hệ thống.

Public subnet: Triển khai các thành phần phải ra/vào Internet như NAT Gateway, Load Balancer.

Private subnet: Chạy ứng dụng backend, EC2, ECS hoặc EKS worker tránh phơi bày trực tiếp ra Internet.

DB subnet: Tách biệt hoàn toàn, chỉ cho phép truy cập từ private subnet, rất an toàn khi dùng Amazon RDS/MongoDB/Redis, v.v.

Độ sẵn sàng và Bảo mật
Mỗi AZ đều có đủ cả 3 subnet giúp bạn xây dựng kiến trúc Multi-AZ/high availability.

Đặt security group và NACL riêng cho từng loại subnet tăng tính bảo mật.

RDS/Elasticache Multi-AZ chỉ hoạt động đúng nếu có subnet group trải đều các AZ như trên.

---
Template tạo VPC1
```
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0
AWSTemplateFormatVersion: '2010-09-09'
Description: Template for building a sample VPC with subnets and a prefix list shared to member accounts in an AWS Organization. 

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: VPC Configuration Details
        Parameters: 
            - vpcCIDR
            - vpcAccountIds
            - publicTierAccountIds
            - privateTierAccountIds
            - dbTierAccountIds
      - Label:
          default: Required Parameters
        Parameters:
            - s3Bucket
            - orgArn
            - env
    ParameterLabels:
      vpcCIDR:
        default: IP Range for VPC
      vpcAccountIds:
        default: Accounts that will access the shared VPC
      publicTierAccountIds:
        default: Accounts with Public Subnet Access
      privateTierAccountIds:
        default: Accounts with Private Subnet Access
      dbTierAccountIds:
        default: Accounts with Database Subnet Access
      s3Bucket:
        default: S3 template bucket
      orgArn:
        default: Amazon Resource Number (ARN) for the AWS Organization
      env:
        default: SDLC Environment of this VPC

Parameters:
  s3Bucket:
    Type: String
    Description: S3 bucket holding template files
  vpcAccountIds:
    Type: String
    Description: Comma delimited list of Account ID requiring shared VPC access
  publicTierAccountIds:
    Type: String
    Description: Comma delimited list of Account ID requiring Public Subnet access
  privateTierAccountIds:
    Type: String
    Description: Comma delimited list of Account ID requiring Private Subnet access
  dbTierAccountIds:
    Type: String
    Description: Comma delimited list of Account ID requiring Database Subnet access
  vpcCIDR:
    Type: String
    Description: IP Range in CIDR format (max /16)
    AllowedPattern: '^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/([0-9]|[1-2][0-9]|3[0-2]))$'
  orgArn:
    Type: String
    Description: AWS Organization ARN
    AllowedPattern: '^arn:aws:organizations::\d{12}:organization\/o-[a-z0-9]{10,32}'
  env:
    Type: String
    Description: SDLC Environment
    AllowedValues:
      - Dev
      - Test
      - Prod

Resources:
  # Provides a list of IP ranges that can be used in security groups and routing tables
  # to provide access from on-premises networks
  OnPremPrefixList:
    Type: AWS::EC2::PrefixList
    Properties:
      PrefixListName: "on-prem-networks"
      AddressFamily: "IPv4"
      MaxEntries: 10
      Entries:
        - Cidr: "10.100.1.0/24" # This value can be moved to a parameter for reusability 
          Description: "Charlotte Office"
        - Cidr: "10.100.2.0/26"
          Description: "Seattle Office"
      Tags:
        - Key: "Name"
          Value: "Ops team networks"
        - Key: Environment
          Value: !Ref env
  
  # Construct your VPC components here.  Note we are using the intrisic function !CIDR here
  # to subnet our VPC range.  You could also assign this value via a parameter to provide more 
  # flexbility in subnet sizing.
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: !Ref vpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      InstanceTenancy: default
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'vpc'
        - Key: Environment
          Value: !Ref env
  PublicSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [ 0, !Cidr [ !Ref vpcCIDR, 6, 4 ]]
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select 
        - 0
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'public-subnet1'
        - Key: Environment
          Value: !Ref env
  PublicSubnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [ 1, !Cidr [ !Ref vpcCIDR, 6, 4 ]]
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select 
        - 1
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'public-subnet2'
        - Key: Environment
          Value: !Ref env
  PrivateSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [ 2, !Cidr [ !Ref vpcCIDR, 6, 4 ]]
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select 
        - 0
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'private-subnet1'
        - Key: Environment
          Value: !Ref env
  PrivateSubnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [ 3, !Cidr [ !Ref vpcCIDR, 6, 4 ]]
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select 
        - 1
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'private-subnet2'
        - Key: Environment
          Value: !Ref env
  DbSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [ 4, !Cidr [ !Ref vpcCIDR, 6, 4 ]]
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select 
        - 0
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'db-subnet1'
        - Key: Environment
          Value: !Ref env
  DbSubnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [ 5, !Cidr [ !Ref vpcCIDR, 6, 4 ]]
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select 
        - 1
        - !GetAZs 
          Ref: 'AWS::Region'
      Tags:
        - Key: Name
          Value: !Join 
            - '-'
            - - !Ref 'AWS::StackName'
              - 'db-subnet2'
        - Key: Environment
          Value: !Ref env
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'
  InternetGatewayAttachment:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties: 
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC
  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties: 
      Tags: 
        - Key: Name
          Value: PublicRouteTable
      VpcId: !Ref VPC
  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties: 
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
      RouteTableId: !Ref PublicRouteTable
  PublicSubnet1RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties: 
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1
  PublicSubnet2RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties: 
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  # Share resources to target accounts
  PublicSubnetsShare:
    Type: "AWS::RAM::ResourceShare"
    DependsOn:
        - PublicSubnet1Id
        - PublicSubnet2Id
    Properties:
      Name: "Public Subnet Shares"
      ResourceArns:
        - !Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PublicSubnet1}'
        - !Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PublicSubnet2}'
      Principals: !Split [ ",", !Ref publicTierAccountIds ]
      Tags:
        - Key: "Environment"
          Value: !Ref env
  PrivateSubnetsShare:
    Type: "AWS::RAM::ResourceShare"
    DependsOn:
      - PrivateSubnet1Id
      - PrivateSubnet2Id
    Properties:
      Name: "Private Subnet Shares"
      ResourceArns:
        - !Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PrivateSubnet1}'
        - !Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PrivateSubnet2}'
      Principals: !Split [ ",", !Ref privateTierAccountIds ]
      Tags:
        - Key: "Environment"
          Value: !Ref env
  DbSubnetsShare:
    Type: "AWS::RAM::ResourceShare"
    DependsOn:
      - DbSubnet1Id
      - DbSubnet2Id
    Properties:
      Name: "Database Subnet Shares"
      ResourceArns:
        - !Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${DbSubnet1}'
        - !Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${DbSubnet2}'
      Principals: !Split [ ",", !Ref dbTierAccountIds ]
      Tags:
        - Key: "Environment"
          Value: !Ref env
  OnPremPrefixListShare:
    Type: "AWS::RAM::ResourceShare"
    Properties:
      Name: "OnPrem Prefix List"
      ResourceArns:
        - !GetAtt OnPremPrefixList.Arn
      Principals: 
        - !Ref orgArn
      Tags:
        - Key: "Environment"
          Value: !Ref env

  # Create SSM Parameters that can be used by member accounts to reference
  # key attribute values
  VpcId:
    Type: AWS::CloudFormation::StackSet
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with the shared VPC ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/Vpc'
        - ParameterKey: Description
          ParameterValue: !Sub '${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref VPC
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref vpcAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-VpcIdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  PublicSubnet1Id:
    Type: AWS::CloudFormation::StackSet
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with PublicSubnet1's ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/PublicSubnet1'
        - ParameterKey: Description
          ParameterValue: !Sub 'Public Subnet 1 ID in ${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref PublicSubnet1
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref publicTierAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-PublicSubnet1IdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  PublicSubnet2Id:
    Type: AWS::CloudFormation::StackSet
    DependsOn: VPC
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with PublicSubnet2's ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/PublicSubnet2'
        - ParameterKey: Description
          ParameterValue: !Sub 'Public Subnet 2 ID in ${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref PublicSubnet2
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref publicTierAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-PublicSubnet2IdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  PrivateSubnet1Id:
    Type: AWS::CloudFormation::StackSet
    DependsOn: VPC
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with PrivateSubnet1's ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/PrivateSubnet1'
        - ParameterKey: Description
          ParameterValue: !Sub 'Private Subnet 1 ID in ${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref PrivateSubnet1
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref privateTierAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-PrivateSubnet1IdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  PrivateSubnet2Id:
    Type: AWS::CloudFormation::StackSet
    DependsOn: VPC
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with PrivateSubnet2's ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/PrivateSubnet2'
        - ParameterKey: Description
          ParameterValue: !Sub 'Private Subnet 2 ID in ${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref PrivateSubnet2
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref privateTierAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-PrivateSubnet2IdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  DbSubnet1Id:
    Type: AWS::CloudFormation::StackSet
    DependsOn: VPC
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with DbSubnet1's ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/DbSubnet1'
        - ParameterKey: Description
          ParameterValue: !Sub 'DB Subnet 1 ID in ${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref DbSubnet1
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref dbTierAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-DbSubnet1IdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  DbSubnet2Id:
    Type: AWS::CloudFormation::StackSet
    DependsOn: VPC
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with DbSubnet2's ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/DbSubnet2'
        - ParameterKey: Description
          ParameterValue: !Sub 'DB Subnet 2 ID in ${env} VPC'
        - ParameterKey: Value
          ParameterValue: !Ref DbSubnet2
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref dbTierAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-DbSubnet2IdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
  OnPremPrefixListId:
    Type: AWS::CloudFormation::StackSet
    DependsOn: VPC
    Properties: 
      AdministrationRoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/NetworkParameterAdminRole
      Description: Publishes a parameter with on prem prefix list ID
      ExecutionRoleName: NetworkParameterExecutionRole
      OperationPreferences: 
          FailureToleranceCount: 1
          MaxConcurrentCount: 2
      Parameters: 
        - ParameterKey: Key
          ParameterValue: !Sub '/Network/${env}/OnPrem-PrefixList'
        - ParameterKey: Description
          ParameterValue: 'Prefix List ID for on-premises networks'
        - ParameterKey: Value
          ParameterValue: !Ref OnPremPrefixList
        - ParameterKey: Type
          ParameterValue: String
        - ParameterKey: Env 
          ParameterValue: !Ref env
      PermissionModel: SELF_MANAGED
      StackInstancesGroup: 
        - DeploymentTargets:
            Accounts: !Split [ "," , !Ref vpcAccountIds ]
          Regions:
            - !Ref 'AWS::Region'
      StackSetName: !Sub '${env}-OnPremPrefixListIdParameter'
      Tags: 
        - Key: Environment
          Value: !Ref env
      TemplateURL: !Sub https://${s3Bucket}.s3.amazonaws.com/ssm-parameter-stackset.yaml
```
Template tạo VPC2
```
AWSTemplateFormatVersion: '2010-09-09'
Description: EKS cluster using a VPC with two public subnets

Parameters:

  NumWorkerNodes:
    Type: Number
    Description: Number of worker nodes to create
    Default: 3

  WorkerNodesInstanceType:
    Type: String
    Description: EC2 instance type for the worker nodes
    Default: t2.small

  KeyPairName:
    Type: String
    Description: Name of an existing EC2 key pair (for SSH-access to the worker node instances)

Mappings:

  VpcIpRanges:
    Option1:
      VPC: 10.0.0.0/16       # 00001010.00000000.xxxxxxxx.xxxxxxxx
      Subnet1: 10.0.0.0/18   # 00001010.00000000.00xxxxxx.xxxxxxxx
      Subnet2: 10.0.64.0/18  # 00001010.00000000.01xxxxxx.xxxxxxxx

  # IDs of the "EKS-optimised AMIs" for the worker nodes:
  # https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
  EksAmiIds:
    us-east-1:
      Standard: ami-0a0b913ef3249b655
    us-east-2:
      Standard: ami-0958a76db2d150238
    us-west-2:
      Standard: ami-0f54a2f7d2e9c88b3
    eu-west-1:
      Standard: ami-00c3b2d35bddd4f5c

Resources:

  #============================================================================#
  # VPC
  #============================================================================#

  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !FindInMap [ VpcIpRanges, Option1, VPC ]
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !FindInMap [ VpcIpRanges, Option1, Subnet1 ]
      AvailabilityZone: !Select
        - 0
        - !GetAZs ""
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-Subnet1"

  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !FindInMap [ VpcIpRanges, Option1, Subnet2 ]
      AvailabilityZone: !Select
        - 1
        - !GetAZs ""
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-Subnet2"

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Ref AWS::StackName

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-PublicSubnets"

  InternetGatewayRoute:
    Type: AWS::EC2::Route
    # DependsOn is mandatory because route targets InternetGateway
    # See here: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-dependson.html#gatewayattachment
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  Subnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Subnet1
      RouteTableId: !Ref RouteTable

  Subnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Subnet2
      RouteTableId: !Ref RouteTable
```
