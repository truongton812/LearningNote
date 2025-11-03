# RDS

RDS Oracle instance là một thực thể cơ sở dữ liệu riêng biệt do dịch vụ Amazon RDS quản lý, chạy engine Oracle trong môi trường đám mây AWS. Mỗi instance có cấu hình tài nguyên riêng (CPU, RAM, storage) và là nơi lưu trữ dữ liệu cũng như xử lý các truy vấn từ ứng dụng.​

Một RDS instance là một cơ sở dữ liệu hoạt động độc lập mà bạn có thể kết nối trực tiếp để đọc và ghi dữ liệu.

Nó không giống với Read Replica. Read Replica là bản sao chỉ đọc của một instance chính, dùng để xử lý các truy vấn chỉ đọc, giảm tải cho instance chính và giúp tăng khả năng mở rộng đọc. Read Replica sao chép dữ liệu từ instance chính một cách không đồng bộ.​

Tóm lại, instance là thực thể cơ sở dữ liệu chính (có thể đọc và ghi), còn read replicas là các bản sao phụ chỉ để đọc, hỗ trợ phân phối tải đọc hiệu quả hơn.

Trong mô hình chuẩn, một database Oracle trong RDS thường chỉ chạy trên một instance chính.

Để mở rộng hoặc tăng tính sẵn sàng, người dùng có thể tạo nhiều instance như read replicas hoặc standalone cho các mục đích như phân tải đọc, backup hoặc failover.

Scale trong AWS RDS Oracle:

- Scale đọc (Read Replica): Hỗ trợ tạo các read replicas để phân tải truy vấn đọc và tăng khả năng mở rộng đọc của hệ thống.​

- Vertical Scaling: Tăng cường tài nguyên cho instance như CPU, RAM, và lưu trữ thông qua thao tác thủ công hoặc tự động, phù hợp cho các tình huống cần tăng hiệu năng ngay lập tức.​
