EC2

Là dịch vụ máy chủ ảo của AWS

Khởi tạo EC2 sẽ sinh ra 1 primary ENI gắn vào EC2 đó. (Nếu stop EC2 → primary ENI cũng bị xóa theo). Primary ENI chứa các thông tin mạng:

- Primary private IP (bắt buộc). IP này sẽ show ra khi dùng lệnh “ip addr" và nó nằm ở trong VPC, chỉ truy cập nội bộ VPC.

- Public IP (nếu chọn "Auto assign public IP" khi tạo EC2). IP này là IP NAT để map từ public IP vào primary private IP. IP này sẽ không show ra khi dùng lệnh "ip addr", phải curl về metadata để biết (http://169.254.169.254/latest/meta-data/public-ipv4)

- Elastic IP (nếu gán)

- SG

- MAC Address

Lưu ý: EC2 và ENI là AZ-level resources
