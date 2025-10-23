ELB bound với VPC

Target group là gì

Target Group trong AWS Load Balancer là một tập hợp các đích (mục tiêu) mà Load Balancer phân phối lưu lượng truy cập đến. Mục tiêu có thể là các EC2 instances, địa chỉ IP, Lambda function hoặc các Load Balancer khác.

Vai trò của Target Group:
- Target Group giúp tổ chức và quản lý các backend server (ví dụ EC2 instance) phục vụ ứng dụng.

- Load Balancer sẽ route (điều hướng) lưu lượng mạng đến các target trong nhóm này dựa trên quy tắc và thuật toán cân bằng tải.

- Target Group còn thực hiện kiểm tra sức khỏe (Health Check), để chỉ gửi traffic đến các mục tiêu đang hoạt động tốt.

Lưu ý: option stickiness được enable/disable trong target group

Trong quá trình tạo target group trên AWS, hai trường protocol và port có những ý nghĩa quan trọng trong việc xác định cách load balancer phân phối lưu lượng đến các mục tiêu (targets).

Protocol
Protocol xác định giao thức mà target group sẽ sử dụng để giao tiếp với các mục tiêu (ví dụ: EC2 instances hoặc ECS tasks).

Các lựa chọn phổ biến là HTTP, HTTPS, TCP, hoặc TLS.

Ví dụ, nếu chọn HTTP, thì load balancer sẽ gửi các yêu cầu từ phía khách hàng qua giao thức HTTP đến các mục tiêu trong target group.

Port
Port chỉ định cổng mà các mục tiêu trong target group sẽ lắng nghe để nhận lưu lượng.

Port có thể là một số cố định (ví dụ: 80 hoặc 443), hoặc hoàn toàn phù hợp với cấu hình của ứng dụng mục tiêu.

Ví dụ, nếu đặt Port là 80, thì các EC2 instances hoặc ECS tasks trong target group phải nghe trên cổng này để nhận các yêu cầu từ load balancer.

---

Option "Availability Zones and subnets" khi tạo Application Load Balancer trên AWS cho phép chọn khu vực khả dụng (AZ - Availability Zone) và các subnet tương ứng mà node Load Balancer sẽ được đặt trong đó


mặc dù về mặt kiến trúc, một AZ có thể có nhiều subnet, thì trong quá trình tạo và cấu hình ALB, bạn chỉ được phép chọn một subnet cho mỗi AZ để ALB tạo ra các node load balancer trong đó

Khi tạo Application Load Balancer (ALB) trên AWS, AWS sẽ tạo một hoặc nhiều node load balancer ở mỗi Availability Zone (AZ) bạn chọn. Mỗi node này được đại diện bằng một Elastic Network Interface (ENI) trong subnet mà bạn đã chọn ở AZ đó.

Elastic Network Interface (ENI) là một thành phần mạng logic trong VPC, tương tự như một card mạng ảo (virtual NIC). ENI là một interface mạng logic được gán vào một subnet cụ thể trong VPC.


Khi bạn chọn một subnet trong một AZ để tạo ALB, AWS sẽ tạo một ENI trong subnet đó. ENI này có địa chỉ IP trong dải subnet và mang nhiệm vụ nhận và truyền tất cả lưu lượng mạng (traffic) đến/đi từ node load balancer trong AZ đó.

Mỗi node load balancer tương ứng với một ENI riêng biệt trong subnet đã chọn, làm điểm xử lý mạng cho ALB trong vùng AZ đó.

ENI này giữ vai trò như gateway nhận request từ client và chuyển tiếp request đến các target (ví dụ EC2 instance) trong target group.

Lưu ý ELB sẽ chỉ forward traffic đến AZ nào được đặt node load balancer (và node load balancer phải đặt trong public subnet nếu ELB muốn nhận traffic từ internet)

---

Khi tạo ELB với internet-facing thì phải trỏ về public subnet (subnet có route đi ra internet gateway) thì ELB mới nhận được internet traffic

tôi thấy trong phần "Availability Zones and subnets" khi tạo ALB có mô tả như sau: "Select at least two Availability Zones and a subnet for each zone. A load balancer node will be placed in each selected zone and will automatically scale in response to traffic. The load balancer routes traffic to targets in the selected Availability Zones only." 

-> Khi tạo ALB dưới dạng internet-facing (nhận traffic từ internet), bạn bắt buộc phải chọn subnet có route đến Internet Gateway (tức public subnet) cho placement của ALB. Nếu chọn private subnet, AWS cảnh báo rằng load balancer sẽ không nhận được traffic từ internet do không có route đến IGW.

Lưu ý: Cảnh báo này chỉ áp dụng cho node của ALB (ENI của ALB) – chứ không phải các target (EC2/Fargate) phía sau.

Các target (instance, IP, Lambda, container,...) nằm trong private subnet vẫn nhận được traffic bình thường từ ALB, vì load balancer sẽ forward traffic nội bộ qua local VPC routing – không bắt buộc phải public subnet

---

Security Group (SG) cho Application Load Balancer (ALB) là một thành phần tường lửa ảo giúp kiểm soát lưu lượng mạng vào và ra của chính ALB.

SG của ALB kiểm soát các kết nối đến từ client (người dùng hoặc dịch vụ ngoài).

Ví dụ, SG của ALB thường mở các port phổ biến như HTTP (80), HTTPS (443) để nhận lưu lượng truy cập web.

ALB sẽ chấp nhận hoặc từ chối các yêu cầu dựa trên quy tắc SG này.

Mối liên hệ giữa Security Group của ALB và Security Group của EC2

EC2 instance thường có SG riêng riêng biệt, kiểm soát lưu lượng Vào/Ra từ instance đó.

Khi ALB chuyển tiếp traffic đến các EC2 (thường qua Target Group), các request này được thực hiện giữa ALB và các EC2 trong mạng nội bộ. Lưu ý cần cấu hình Security Group cho EC2 để chấp nhận traffic từ ALB, chứ Không cấu hình cho target group, vì target group chỉ là tập hợp các target (EC2), không phải entity có SG riêng.

Do đó, để EC2 nhận traffic từ ALB, SG của EC2 phải cho phép nhận lưu lượng từ SG của ALB (chấp nhận inbound traffic từ SG của ALB).

Nói cách khác, SG của EC2 cần có rule inbound cho phép traffic đến từ SG của ALB, thường là cho các port ứng dụng mà EC2 đang chạy (ví dụ port 80 cho web).

---


mỗi Load Balancer node của NLB thường có một hoặc nhiều IP tĩnh riêng biệt, khác với ALB thường không có IP tĩnh mà sử dụng DNS dynamic.

NLB tạo các node load balancer nằm tại các Availability Zone (AZ) mà bạn chọn.

Mỗi node NLB trong mỗi AZ sẽ được cấp IP tùy thuộc vào cách cấu hình:
- nếu chọn internet facing NLB thì node NLB sẽ có ip public (do AWS cấp hoặc ta có thể tự gán EIP). Lưu ý nếu sử dụng dạng internet facing NLB thì subnet phải là public subnet, nếu là private sẽ không nhận được traffic
- nếu chọn internal NLB thì node sẽ chỉ được gán private ip


DNS của loại internet facing NLB được ánh xạ tới các địa chỉ IP public của các node trong các AZ. Khi client truy cập, DNS phân giải dựa trên địa lý và trạng thái để đưa client đến node phù hợp.



Lợi ích khi dùng NLB trỏ tới ALB:
- Hỗ trợ IP tĩnh và Elastic IP (EIP): NLB cho phép bạn có IP tĩnh hoặc Elastic IP trên mỗi AZ, thuận tiện cho các hệ thống hoặc firewall, DNS yêu cầu IP cố định. ALB mặc định chỉ có DNS name và IP động, nên nếu cần IP tĩnh thì đặt NLB trước ALB là tốt.

- Kết nối ở Layer 4 với throughput cao và độ trễ thấp: NLB xử lý ở lớp 4 với TCP/UDP, giúp xử lý lượng lớn kết nối nhanh và bền bỉ. Khi làm frontend cho ALB (làm backend), NLB có thể tiếp nhận traffic khổng lồ một cách hiệu quả, rồi chuyển tới ALB ở Layer 7 xử lý logic nghiệp vụ.

- Giảm tải cho ALB: NLB có thể chịu nhiều kết nối đến với hiệu suất rất cao, giúp giảm bớt áp lực trực tiếp cho ALB, giữ cho ALB tập trung xử lý các logic tầng ứng dụng như routing, authentication, cookie.

Lưu ý:
- Network Load Balancer (NLB) không có một IP tĩnh duy nhất chung cho toàn bộ Load Balancer. Thay vào đó: Mỗi Load Balancer node của NLB tồn tại trong một Availability Zone (AZ) và được cấp một hoặc nhiều địa chỉ IP tĩnh riêng biệt. Các IP tĩnh này có thể là địa chỉ IP private trong subnet tương ứng, hoặc là Elastic IP (EIP) nếu bạn gán IP public cho NLB Internet-facing.

- Nếu bạn có NLB forward traffic đến 3 AZ, Bạn cần whitelist từng IP tĩnh của từng node NLB trong mỗi AZ. Nghĩa là nếu NLB hoạt động trên 3 AZ, bạn sẽ có ít nhất 3 IP tĩnh (một cho mỗi AZ), hoặc nhiều hơn nếu mỗi node có nhiều IP tĩnh. Bạn không chỉ whitelist một IP duy nhất, mà phải bao gồm đủ tất cả các IP tĩnh tương ứng với các node ở từng AZ của NLB.

---

CloudFormation template để tạo ALB

```
AWSTemplateFormatVersion: '2010-09-09'
Description: Mẫu CloudFormation tạo ALB kết hợp Auto Scaling Group

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID để tạo resources
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Danh sách subnet cho ALB và Auto Scaling Group
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: KeyPair để truy cập instance
  InstanceType:
    Type: String
    Default: t3.micro
    Description: Loại instance EC2

Resources:
  MyLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: MyALB
      Subnets: !Ref SubnetIds
      Scheme: internet-facing
      Type: application

  MyTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: MyTargetGroup
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VpcId
      HealthCheckPath: /
      Matcher:
        HttpCode: 200
      TargetType: instance

  MyListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref MyLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref MyTargetGroup

  LaunchConfig:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: ami-0c55b159cbfafe1f0  # AMI Amazon Linux 2 (thay theo region của bạn)
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SecurityGroups: []  # Thêm nếu cần

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier: !Ref SubnetIds
      LaunchConfigurationName: !Ref LaunchConfig
      MinSize: 1
      MaxSize: 3
      DesiredCapacity: 1
      TargetGroupARNs:
        - !Ref MyTargetGroup
      Tags:
        - Key: Name
          Value: MyASGInstance
          PropagateAtLaunch: true
```

Đúng vậy, khi bạn dùng Application Load Balancer (ALB) kết hợp với Auto Scaling Group (ASG), thì ASG cần phải được khai báo trỏ đến target group của ALB.

Điều này có nghĩa là:

ASG không đính trực tiếp với ALB mà đính với target group (nhóm đích) mà ALB dùng để cân bằng tải.

Khi ASG tạo hoặc huỷ instance, AWS sẽ tự động đăng ký hoặc gỡ các instance này vào target group để cân bằng tải qua ALB.

Việc liên kết này cũng cho phép ASG sử dụng các health check của target group để quyết định duy trì hay thay thế instance không khỏe.

