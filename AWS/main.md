### Thiết kế mô hình ứng dụng 3 lớp trên aws

Kiến trúc ứng dụng 3 lớp trên AWS gồm Presentation (UI), Business Logic và Data, được triển khai bằng các dịch vụ AWS như EC2, RDS, S3 và Load Balancer để tối ưu hóa bảo mật, hiệu năng và khả năng mở rộng.

Tổng quan mô hình 3 lớp

![alt text](Image/01.%20aws%203%20tiers%20application.png)


Tầng Presentation (UI): Giao diện người dùng, nhận thao tác đầu vào và hiển thị dữ liệu; thường được deploy lên Amazon S3 (static web) hoặc EC2/Elastic Beanstalk cho web động (Node.js, PHP, Python...) kết hợp với Amazon CloudFront tăng tốc phân phối và Load Balancer để phân phối tải và tăng khả dụng UI.

Tầng Business Logic: Xử lý nghiệp vụ, xác thực, điều phối luồng dữ liệu giữa UI và Data; có thể triển khai trên EC2 (servers), ECS/ EKS (container microservices), hoặc Lambda (serverless functions). Sử dụng Auto Scaling Group để tăng scale

Tầng Data: Lưu trữ, truy vấn dữ liệu, sử dụng Amazon RDS/Aurora (quan hệ) hoặc DynamoDB (phi quan hệ), kết hợp S3 lưu file hoặc backup.







##### Kiến trúc mạng và bảo mật

Sử dụng VPC riêng biệt, chia subnet (public/private) để kiểm soát truy cập. Application Load Balancer nằm ở public subnet, EC2/ECS/EKS/BLogic nằm ở private subnet, DB/Data chỉ truy cập nội bộ hoặc qua bảo mật.

Tích hợp Auto Scaling, IAM, nhóm bảo mật Security Group, Network ACL, AWS WAF để nâng cao khả năng bảo vệ ứng dụng, truy cập đúng lớp.

Định tuyến thông tin qua các tầng, cách ly truy cập trực tiếp database từ client, giảm rủi ro bảo mật.


##### Quy trình triển khai trên AWS

- Tạo VPC, chia các subnet.
- Tạo Internet Gateway (IGW) để kết nối mạng giữa VPC và Internet. Nó cho phép các instance hoặc tài nguyên trong VPC có thể gửi và nhận lưu lượng mạng từ bên ngoài Internet. Internet Gateway được gán (attached) với toàn bộ VPC, không phải với từng subnet riêng biệt. Tuy nhiên, chỉ những subnet được cấu hình route tới IGW trong bảng định tuyến mới có khả năng truy cập Internet trực tiếp, gọi là subnet public. Subnet private không có route tới IGW mà thường truy cập Internet qua NAT Gateway hoặc NAT Instance.
- A Network Address Translation (NAT) Gateway is used to allow resources within a private subnet in a Virtual Private Cloud (VPC) to access the internet while keeping them hidden and protected from direct access. It’s important to note that a NAT gateway must always be launched in a public subnet.
- Deploy UI (frontend) lên S3 hoặc EC2, sử dụng CloudFront/CDN.
- Deploy Logic/backend trên EC2/ECS/Lambda theo specs nghiệp vụ, kết nối API Gateway nếu cần.
- Tạo database, cấu hình RDS/DynamoDB/S3 theo loại dữ liệu xử lý. Để kiểm soát truy cập đến database We need a security group so that other applications inside our VPC can connect to our database instance. You can either create a security group for your instance or use an existing one. 
- Thiết lập Load Balancer, routing request (UI → Logic → Data).
- Triển khai bảo mật, thiết lập IAM, WAF, Security Group, Auto Scaling.


