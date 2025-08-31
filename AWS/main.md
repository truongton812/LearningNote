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


### Mô hình 3 lớp thiết kế dùng API Gateway trên AWS

Mục đích của mô hình dùng API Gateway có thể linh hoạt tùy theo thiết kế ứng dụng, gồm hai trường hợp chính:

- Cho người dùng Internet truy cập trực tiếp: API Gateway làm điểm đầu mối nhận request từ client như web, mobile app hoặc ứng dụng bên ngoài, sau đó định tuyến, xác thực và chuyển tiếp yêu cầu đến backend. Đây là cách phổ biến để phục vụ người dùng cuối qua API, đồng thời tận dụng khả năng bảo mật, giới hạn tần suất và quản lý API của API Gateway.

Người dùng vẫn truy cập ứng dụng qua web browser hoặc app trên điện thoại như bình thường.

Tuy nhiên, khi người dùng tương tác (ví dụ bấm nút, gửi form), frontend (giao diện web hoặc app) không gọi trực tiếp backend mà sẽ gửi các API requests tới API Gateway.

API Gateway đóng vai trò như một “cổng” trung gian nhận các yêu cầu đó, thực hiện các bước như:

    + Xác thực (kiểm tra người dùng hợp lệ, token,...).

    + Ủy quyền (phân quyền truy cập).

    + Giới hạn tần suất để tránh quá tải (rate limiting).

    + Định tuyến request đến đúng service backend (ví dụ Lambda, EC2, container).

    + Backend xử lý yêu cầu rồi API Gateway truyền kết quả trả về frontend.

    + Frontend nhận dữ liệu và hiển thị cho người dùng trên trình duyệt.

- Phục vụ các hệ thống hoặc dịch vụ nội bộ khác gọi vào: API Gateway cũng có thể được dùng trong kiến trúc microservices làm điểm trung gian để các service nội bộ gọi nhau, giúp quản lý API nội bộ, bảo mật, logging và kiểm soát truy cập. Trường hợp này API Gateway không công khai ra Internet, chỉ nội bộ hoặc trong mạng riêng.


Mô hình 3 lớp thiết kế dùng API Gateway trên AWS gồm các tầng: Presentation (UI), Business Logic và Data, với API Gateway đóng vai trò làm điểm trung gian quản lý, bảo mật và định tuyến các API request từ client đến backend services.

Tổng quan mô hình 3 lớp có API Gateway

Tầng Presentation (UI): Client (web, mobile) gửi request đến API Gateway thay vì gọi trực tiếp backend.

API Gateway: Nhận yêu cầu, thực hiện xác thực, ủy quyền, giới hạn tần suất truy cập, chuyển đổi dữ liệu, và định tuyến yêu cầu đến các service backend tương ứng (có thể là Lambda functions, EC2, ECS, hoặc ALB).

Tầng Business Logic: Các service backend xử lý nghiệp vụ, có thể triển khai trên Lambda hoặc containerized services trên ECS/EKS hoặc máy chủ EC2.

Tầng Data: Database (RDS, DynamoDB, v.v.) nhận kết nối từ tầng business logic, không công khai trực tiếp ra client.

Lợi ích API Gateway trong mô hình 3 lớp

Cung cấp điểm truy cập duy nhất cho các API, giúp quản lý tập trung và bảo mật tốt hơn.

Hỗ trợ các chính sách xác thực và phân quyền linh hoạt với IAM, Cognito, hoặc các custom authorizer.

Tối ưu routing request theo đường dẫn (path-based routing) đến service backend đa dạng.

Có thể tích hợp cache để tăng hiệu năng, tiết kiệm tải backend.

Giúp tách biệt front-end và back-end rõ ràng, dễ mở rộng, vận hành, và bảo trì.

Phương án triển khai

Khai báo API và resource trên Amazon API Gateway: định nghĩa các endpoints, methods.

Tích hợp backend qua nhiều kiểu integration: AWS Lambda, HTTP backend (EC2, ECS qua ALB), AWS service proxy.

Áp dụng chính sách bảo mật như IAM, authorizer (JWT, Lambda authorizer) trên API Gateway.

Backend service xử lý nghiệp vụ, gọi DB, thực thi logic.

Lớp dữ liệu quản lý bằng các dịch vụ RDS, DynamoDB, S3 (lưu trữ file).

Chức năng Auto Scaling, monitoring qua CloudWatch đảm bảo ổn định.
