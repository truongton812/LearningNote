### Thiết kế mô hình ứng dụng 3 lớp trên aws

Kiến trúc ứng dụng 3 lớp trên AWS gồm Presentation (UI), Business Logic và Data, được triển khai bằng các dịch vụ AWS như EC2, RDS, S3 và Load Balancer để tối ưu hóa bảo mật, hiệu năng và khả năng mở rộng.

Tổng quan mô hình 3 lớp
Tầng Presentation (UI): Giao diện người dùng, nhận thao tác đầu vào và hiển thị dữ liệu; thường được deploy lên Amazon S3 (static web) hoặc EC2/Elastic Beanstalk cho web động, kết hợp với Amazon CloudFront tăng tốc phân phối.

Tầng Business Logic: Xử lý nghiệp vụ, xác thực, điều phối luồng dữ liệu giữa UI và Data; có thể triển khai trên EC2 (servers), ECS (container microservices), hoặc Lambda (serverless functions).

Tầng Data: Lưu trữ, truy vấn dữ liệu, sử dụng Amazon RDS/Aurora (quan hệ) hoặc DynamoDB (phi quan hệ), kết hợp S3 lưu file hoặc backup.

Các dịch vụ AWS triển khai từng lớp
Presentation:

S3 + CloudFront: Static web site, phân phối CDN.

EC2/Elastic Beanstalk: Web server động (Node.js, PHP, Python...).

Load Balancer (ALB): Phân phối tải và tăng khả dụng UI.

Business Logic:

EC2 instance, Auto Scaling Group: Hạ tầng máy chủ.

ECS/EKS: Microservices kiến trúc container hóa.

Lambda: Xử lý serverless, giảm cost.

Data:

RDS/Aurora: Database quan hệ (MySQL, PostgreSQL, SQL Server).

DynamoDB: Database phi quan hệ key-value.

S3: Lưu trữ tài liệu, media.

Kiến trúc mạng và bảo mật
Sử dụng VPC riêng biệt, chia subnet (public/private) để kiểm soát truy cập. Application Load Balancer nằm ở public subnet, EC2/ECS/EKS/BLogic nằm ở private subnet, DB/Data chỉ truy cập nội bộ hoặc qua bảo mật.

Tích hợp Auto Scaling, IAM, nhóm bảo mật Security Group, Network ACL, AWS WAF để nâng cao khả năng bảo vệ ứng dụng, truy cập đúng lớp.

Định tuyến thông tin qua các tầng, cách ly truy cập trực tiếp database từ client, giảm rủi ro bảo mật.

Quy trình triển khai trên AWS
Tạo VPC, chia các subnet.

Deploy UI (frontend) lên S3 hoặc EC2, sử dụng CloudFront/CDN.

Deploy Logic/backend trên EC2/ECS/Lambda theo specs nghiệp vụ, kết nối API Gateway nếu cần.

Tạo database, cấu hình RDS/DynamoDB/S3 theo loại dữ liệu xử lý.

Thiết lập Load Balancer, routing request (UI → Logic → Data).

Triển khai bảo mật, thiết lập IAM, WAF, Security Group, Auto Scaling.

Triển khai CI/CD với CodePipeline, CodeBuild hoặc sử dụng CloudFormation/IaC để tự động hóa.

