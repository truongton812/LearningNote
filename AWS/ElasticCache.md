<img width="974" height="477" alt="image" src="https://github.com/user-attachments/assets/57a32f03-1ec3-4e30-b3c5-b1b958233d8f" />

Trong kiến trúc hệ thống AWS, ElastiCache đóng vai trò là lớp cache nằm giữa ứng dụng (application layer) và cơ sở dữ liệu chính (như RDS, DynamoDB), giúp lưu trữ dữ liệu truy cập thường xuyên trong bộ nhớ để giảm tải và tăng tốc độ đọc. Cache nhận request từ ứng dụng qua endpoint (VPC endpoint cho serverless hoặc trực tiếp cho node-based), xử lý qua proxy layer (nếu serverless) rồi trả dữ liệu nhanh chóng mà không cần query database mỗi lần

Ứng dụng backend (như EC2, Lambda, EKS pods) gọi trực tiếp vào ElastiCache qua endpoint, không phải user gọi trực tiếp vì cache chỉ là internal service trong VPC, không expose public endpoint. User chỉ tương tác với frontend/API Gateway, backend xử lý logic cache miss/hit để tối ưu performance.
​

Luồng gọi điển hình: Backend implement cache pattern: Kiểm tra ElastiCache trước (GET key), nếu miss thì query database (RDS/DynamoDB), lưu kết quả vào cache (SET key value TTL), rồi trả response cho user.
