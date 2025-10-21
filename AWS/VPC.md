# Virutual Private Cloud (VPC)
## I. Định nghĩa và các khái niệm
- Là dịch vụ cho phép bạn khởi chạy các tài nguyên AWS trong mạng ảo cô lập theo logic mà bạn xác định
- VPC là regional service
- Mỗi region luôn có 1 default VPC (172.34.0.0/16) & mỗi AZ luôn có 1 default public subnet (172.31.0.0/20).
- Ngoài default VPC ta có thể tạo các custom VPC. Có thể tạo tối đa 5 custom VPC trong 1 region, mỗi VPC có thể span qua nhiều AZ. Khi tạo custom VPC cần cung cấp 1 IP range (CIDR). CIDR phải là dải IP private của lớp A, B, C. Kích thước CIDR block có thể từ /16 (65.536 IP) đến /28 (16 IP).
- Lưu ý về CIDR:
  - Không được overlap giữa các VPC nếu muốn dùng VPC peering.
  - Không thể tăng/giảm size của CIDR sau khi tạo.
- Subnet là mạng con chia từ VPC CIDR, 1 subnet chỉ gắn liền với 1 AZ, tuy nhiên 1 AZ có thể có nhiều subnet. Thông thường 1 AZ có 2 subnet (public subnet & private subnet).
- Lưu ý: khi tạo subnet trong default VPC mà không explicit associate với route table nào thì mặc định sẽ là public subnet (do khi ta không gán với route table nào thì subnet đấy sẽ được implicit associate với default route table, mà default route table luôn trỏ 0.0.0.0 về internet gateway).
- Routing table: là bảng định tuyến của VPC, bao gồm một tập hợp các rule (được gọi là route), được sử dụng để xác định đường đi, nơi đến của các gói tin từ mạng con hay gateway. Default routing table sẽ gắn với các subnet ko explicitly associate với route table nào.
- Internet Gateway: là tài nguyên thuộc VPC, dùng để kết nối ra internet. 1 VPC chỉ có thể attach với 1 internet gateway
- NAT Gateway
  - Là component giúp EC2 trong private subnet kết nối ra internet.
  - NAT Gateway phải được đặt trong public subnet và phải được gán một Elastic IP (EIP) để có thể giao tiếp ra internet. Khi bạn tạo NAT Gateway, bạn phải chọn một EIP để gán cho nó. NAT Gateway sử dụng EIP này để NAT các kết nối từ private subnet ra internet. Thông qua EIP, các instance trong private subnet được NAT lại IP public tĩnh của NAT Gateway khi truy cập internet. 
  - Lưu lượng từ NAT Gateway khi ra ngoài internet được định tuyến thông qua Internet Gateway. Nếu không có Internet Gateway, NAT Gateway không thể kết nối ra internet mặc dù đã có địa chỉ EIP. Cơ chế routing trong route table của private subnet sẽ chuyển lưu lượng internet ra NAT Gateway, và NAT Gateway tiếp tục chuyển qua Internet Gateway ra ngoài.
  - Cần config route table cho private subnet trỏ 0.0.0.0 về NAT Gateway ⭢ private subnet có thể outbound ra internet (không inbound).
  - NAT Gateway trên AWS được gắn với một Availability Zone (AZ) cụ thể chứ không phải trực tiếp với subnet, nhưng để hoạt động, NAT Gateway phải đặt trong một public subnet thuộc AZ đó. Nếu bạn có nhiều AZ trong một VPC và muốn tăng độ sẵn sàng, bạn nên tạo một NAT Gateway cho mỗi AZ, đặt trong các subnet public tương ứng của từng AZ. Các instance trong private subnet sẽ định tuyến lưu lượng internet ra NAT Gateway trong cùng AZ để giảm độ trễ và tránh rủi ro khi AZ khác bị lỗi.
  - NAT Gateway tính tiền theo giờ + bandwidth
  - NAT Gateway bandwidth là 5GBps, scale up to 100 GBps

## II. Mở rộng VPC
- Ban đầu khi tạo VPC trên AWS, cần phải chỉ định một CIDR block IPv4 duy nhất gọi là "primary CIDR block" cho VPC đó. CIDR này xác định phạm vi địa chỉ IP mà VPC có thể sử dụng. Tuy nhiên, sau khi VPC đã được tạo, ta có thể thêm tối đa 4 "secondary CIDR blocks" nữa, tức là tới 5 CIDR block (giới hạn này có thể tăng lên tối đa 50 theo yêu cầu) cho mỗi VPC. Điều này giúp mở rộng phạm vi địa chỉ IP cho VPC mà không cần tạo thêm VPC mới. Việc này hữu ích khi ta cần thêm nhiều subnet hoặc nhiều tài nguyên hơn. Điều kiện là CIDR block mới thêm phải không overlap với CIDR block hiện có trong VPC hoặc các CIDR đã gán trước đó.
- Khi mở rộng VPC bằng cách thêm CIDR block mới (ví dụ như một secondary CIDR), các subnet tạo trong CIDR mới vẫn sẽ có thể giao tiếp với các subnet trong CIDR chính (primary CIDR), do các route trong bảng định tuyến (route table) mặc định của VPC đều có route "local" cho tất cả CIDR blocks thuộc VPC, nghĩa là các subnet trong các CIDR khác nhau vẫn có thể giao tiếp nội bộ qua route "local" này, trừ khi có chính sách hạn chế hoặc kiểm soát truy cập như security groups và network ACLs.

## III. Mối liên hệ giữa Subnet và Availability Zone (AZ) trong AWS
- Một Availability Zone (AZ) là một vị trí vật lý riêng biệt trong một AWS Region. Mỗi AZ gồm một hoặc nhiều trung tâm dữ liệu (data center) độc lập, giúp tăng tính sẵn sàng và khả năng chịu lỗi cho hệ thống.
- Một Subnet là một mạng con (subnetwork) được tạo trong một VPC và nằm hoàn toàn trong một Availability Zone duy nhất. Nghĩa là, mỗi subnet chỉ gói gọn trong một AZ, không thể kéo dài qua nhiều AZ.
- Khi tạo subnet trong AWS, bạn cần chỉ định CIDR block cho subnet đó và chọn một AZ cụ thể để subnet thuộc về.
- Mỗi VPC có thể có nhiều subnet, mỗi subnet nằm trong một AZ khác nhau nhằm phân bố tài nguyên, hệ thống của bạn có thể được triển khai trên nhiều AZ khác nhau để tăng độ dự phòng và khả năng chịu lỗi.
- Việc phân chia subnet theo AZ giúp AWS và bạn kiểm soát mạng tốt hơn, phân tách tài nguyên theo vùng vật lý, dễ dàng tổ chức kiến trúc dịch vụ phân tán, đảm bảo khi AZ này gặp sự cố thì các subnet (và tài nguyên) ở AZ khác vẫn hoạt động bình thường.
- Khi tạo tài nguyên EC2 trong AWS, bạn cần chỉ định Subnet chứ không phải chỉ định trực tiếp Availability Zone (AZ).
- Mô hình thiết kế 1 VPC
<img width="763" height="355" alt="image" src="https://github.com/user-attachments/assets/ba2a22aa-a743-4c19-aa36-6eef22f9a96f" />
https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html

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


Lưu ý: Các subnet khác AZ trong cùng VPC giao tiếp hoàn toàn bình thường như các subnet trong cùng AZ.



security group bound với vpc

elb bound với vpc, có thể span trên nhiều az

---


nacl là resource của vpc. nacl đc gán với subnet, 1 subnet chỉ đc gán 1 nacl
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
