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

Tạo VPC, chia các subnet.

Deploy UI (frontend) lên S3 hoặc EC2, sử dụng CloudFront/CDN.

Deploy Logic/backend trên EC2/ECS/Lambda theo specs nghiệp vụ, kết nối API Gateway nếu cần.

Tạo database, cấu hình RDS/DynamoDB/S3 theo loại dữ liệu xử lý.

Thiết lập Load Balancer, routing request (UI → Logic → Data).

Triển khai bảo mật, thiết lập IAM, WAF, Security Group, Auto Scaling.

Triển khai CI/CD với CodePipeline, CodeBuild hoặc sử dụng CloudFormation/IaC để tự động hóa.

