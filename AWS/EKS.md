Trong EKS, control plane (gồm API server) và các worker node giao tiếp với nhau chủ yếu qua cổng TCP 443, cổng chuẩn dùng cho Kubernetes API server để giao tiếp an toàn bằng HTTPS.

Các cổng quan trọng cho giao tiếp control plane và worker nodes:

Cổng TCP 443 (Kubernetes API server): worker node và các pod kết nối đến API server để gửi các yêu cầu Kubernetes như tạo pod, cập nhật trạng thái, ...

Cổng TCP 10250 (Kubelet API): control plane kết nối đến kubelet trên các node để lấy logs, thực hiện lệnh từ kubectl attach hay port-forward, ...

Cổng TCP 10256 (kube-proxy): để kube-proxy trên node giao tiếp với các thành phần mạng.

Các cổng NodePort (30000-32767 mặc định): nếu dùng dịch vụ NodePort để expose ngoài.


CloudFormation tạo cụm EKS

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

  #============================================================================#
  # Control plane
  #============================================================================#

  ControlPlane:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Ref AWS::StackName
      Version: "1.10"
      RoleArn: !GetAtt ControlPlaneRole.Arn
      ResourcesVpcConfig:
        SecurityGroupIds:
          - !Ref ControlPlaneSecurityGroup
        SubnetIds:
          - !Ref Subnet1
          - !Ref Subnet2

  ControlPlaneRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
            Effect: Allow
            Principal:
              Service:
                - eks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns: 
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
        - arn:aws:iam::aws:policy/AmazonEKSServicePolicy

  #============================================================================#
  # Control plane security group
  #============================================================================#

  ControlPlaneSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for the elastic network interfaces between the control plane and the worker nodes
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-ControlPlaneSecurityGroup"
      

  ControlPlaneIngressFromWorkerNodesHttps:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Allow incoming HTTPS traffic (TCP/443) from worker nodes (for API server)
      GroupId: !Ref ControlPlaneSecurityGroup
      SourceSecurityGroupId: !Ref WorkerNodesSecurityGroup
      IpProtocol: tcp
      ToPort: 443
      FromPort: 443

  ControlPlaneEgressToWorkerNodesKubelet:
    Type: AWS::EC2::SecurityGroupEgress
    Properties:
      Description: Allow outgoing kubelet traffic (TCP/10250) to worker nodes
      GroupId: !Ref ControlPlaneSecurityGroup
      DestinationSecurityGroupId: !Ref WorkerNodesSecurityGroup
      IpProtocol: tcp
      FromPort: 10250
      ToPort: 10250

  ControlPlaneEgressToWorkerNodesHttps:
    Type: AWS::EC2::SecurityGroupEgress
    Properties:
      Description: Allow outgoing HTTPS traffic (TCP/442) to worker nodes (for pods running extension API servers)
      GroupId: !Ref ControlPlaneSecurityGroup
      DestinationSecurityGroupId: !Ref WorkerNodesSecurityGroup
      IpProtocol: tcp
      FromPort: 443
      ToPort: 443

  #============================================================================#
  # Worker nodes security group
  # Note: default egress rule (allow all traffic to all destinations) applies
  #============================================================================#

  WorkerNodesSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for all the worker nodes
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-WorkerNodesSecurityGroup"
        - Key: !Sub "kubernetes.io/cluster/${ControlPlane}"
          Value: "owned"

  WorkerNodesIngressFromWorkerNodes:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Allow all incoming traffic from other worker nodes
      GroupId: !Ref WorkerNodesSecurityGroup
      SourceSecurityGroupId: !Ref WorkerNodesSecurityGroup
      IpProtocol: "-1"

  WorkerNodesIngressFromControlPlaneKubelet:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Allow incoming kubelet traffic (TCP/10250) from control plane 
      GroupId: !Ref WorkerNodesSecurityGroup
      SourceSecurityGroupId: !Ref ControlPlaneSecurityGroup
      IpProtocol: tcp
      FromPort: 10250
      ToPort: 10250

  WorkerNodesIngressFromControlPlaneHttps:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Allow incoming HTTPS traffic (TCP/443) from control plane (for pods running extension API servers)
      GroupId: !Ref WorkerNodesSecurityGroup
      SourceSecurityGroupId: !Ref ControlPlaneSecurityGroup
      IpProtocol: tcp
      FromPort: 443
      ToPort: 443

  #============================================================================#
  # Worker nodes (auto-scaling group)
  #============================================================================#

  WorkerNodesAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
    Properties:
      LaunchConfigurationName: !Ref WorkerNodesLaunchConfiguration
      MinSize: !Ref NumWorkerNodes
      MaxSize: !Ref NumWorkerNodes
      VPCZoneIdentifier:
        - !Ref Subnet1
        - !Ref Subnet2
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-WorkerNodesAutoScalingGroup"
          PropagateAtLaunch: true
        # Without this tag, worker nodes are unable to join the cluster:
        - Key: !Sub "kubernetes.io/cluster/${ControlPlane}"
          Value: "owned"
          PropagateAtLaunch: true

  WorkerNodesLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    # Wait until cluster is ready before launching worker nodes
    DependsOn: ControlPlane
    Properties:
      AssociatePublicIpAddress: true
      IamInstanceProfile: !Ref WorkerNodesInstanceProfile
      ImageId: !FindInMap
        - EksAmiIds
        - !Ref AWS::Region
        - Standard
      InstanceType: !Ref WorkerNodesInstanceType
      KeyName: !Ref KeyPairName
      SecurityGroups:
        - !Ref WorkerNodesSecurityGroup
      UserData:
        Fn::Base64: !Sub |
            #!/bin/bash
            set -o xtrace
            /etc/eks/bootstrap.sh ${ControlPlane}
            /opt/aws/bin/cfn-signal \
                            --exit-code $? \
                            --stack  ${AWS::StackName} \
                            --resource NodeGroup  \
                            --region ${AWS::Region}

  WorkerNodesInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref WorkerNodesRole

  WorkerNodesRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          Effect: Allow
          Principal:
            Service:
              - ec2.amazonaws.com
          Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

Outputs:

  WorkerNodesRoleArn:
    # Needed for the last step "enable worker nodes to join the cluster":
    # https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html#eks-create-cluster
    Description: ARN of the worker nodes role
    Value: !GetAtt WorkerNodesRole.Arn
```

---

Cách để pod tương tác với tài nguyên trên AWS
- Open ID connect provider
- Pod identity (mới và đơn giản hơn)



Pod Identity (hay EKS Pod Identity) là tính năng AWS cho phép pods trong EKS cluster assume IAM roles trực tiếp mà không cần IAM Roles for Service Accounts (IRSA). Dưới đây là hướng dẫn chi tiết để pod truy xuất S3 bucket.
​

Yêu cầu tiên quyết
EKS cluster version 1.30+ với Pod Identity Agent addon enabled.

AWS CLI v2.15.0+, eksctl v0.166.0+.

kubectl configured với cluster context.
​

bash
# Kiểm tra Pod Identity Agent addon
aws eks describe-addon --cluster-name <cluster-name> --addon-name pod-identity-agent --region <region>
Bước 1: Tạo IAM Role cho Pod
text
# Terraform - IAM Role với Pod Identity trust policy
resource "aws_iam_role" "s3_pod_role" {
  name = "s3-pod-identity-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRoleWithPodIdentity"
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:eks:us-west-2::<account-id>:cluster/<cluster-name>"
      }
      Condition = {
        StringEquals = {
          "k8s.io/pod/namespace" = "default"
          "k8s.io/pod/name"      = "my-s3-pod"
        }
      }
    }]
  })
}

# Attach S3 policy
resource "aws_iam_role_policy_attachment" "s3_policy" {
  role       = aws_iam_role.s3_pod_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}
Bước 2: Tạo PodIdentityAssociation
text
# pod-identity-association.yaml
apiVersion: podidentity.amazonaws.com/v1beta1
kind: PodIdentityAssociation
metadata:
  name: s3-pod-association
  namespace: default
spec:
  clusterName: <your-eks-cluster-name>
  namespace: default
  serviceAccount: my-s3-sa
  roleArn: arn:aws:iam::<account-id>:role/s3-pod-identity-role
bash
kubectl apply -f pod-identity-association.yaml
kubectl wait --for=condition=Established podidentityassociation/s3-pod-association
Bước 3: Deploy Pod với ServiceAccount
text
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-pod
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-pod
  template:
    metadata:
      labels:
        app: s3-pod
    spec:
      serviceAccountName: my-s3-sa  # Phải match với PodIdentityAssociation
      containers:
      - name: app
        image: amazon/aws-cli:latest
        command: ["/bin/sh"]
        args: ["-c", "while true; do aws s3 ls s3://my-bucket --region us-west-2; sleep 30; done"]
bash
# Tạo ServiceAccount trước
kubectl create sa my-s3-sa

kubectl apply -f deployment.yaml
Bước 4: Verify quyền truy xuất
bash
# Kiểm tra pod logs
kubectl logs -f deployment/s3-pod

# Exec vào pod test
kubectl exec -it deployment/s3-pod -- aws sts get-caller-identity
kubectl exec -it deployment/s3-pod -- aws s3 ls s3://my-bucket

##### Bảng so sánh Pod Identity vs IRSA

| Tiêu chí              | Pod Identity (EKS)                  | IRSA (OIDC)                          |
|-----------------------|-------------------------------------|--------------------------------------|
| **Setup complexity**  | Đơn giản (PodIdentityAssociation)  | Phức tạp (OIDC Provider + annotation)|
| **Performance**       | Nhanh (Webhook agent)              | Chậm (Token exchange qua OIDC)      |
| **EKS version**       | 1.30+                              | Tất cả versions                     |
| **Multiple roles/pod**| Hỗ trợ                             | Không hỗ trợ                        |
| **Trust policy**      | `sts:AssumeRoleWithPodIdentity`    | `sts:AssumeRoleWithWebIdentity`     |
| **Cost**              | Miễn phí                           | Miễn phí                            |
| **Scalability**       | Tốt hơn (agent-based)              | Giới hạn token size                 |


Troubleshooting phổ biến
Lỗi "AccessDenied": Kiểm tra namespace/serviceAccount match chính xác trong condition.
​

Association không ready: Đợi PodIdentityAgent rollout hoàn tất.

Role không assume được: Verify cluster ARN format đúng.
​

Pod Identity thay thế IRSA với hiệu suất tốt hơn và setup đơn giản hơn cho EKS workloads.
