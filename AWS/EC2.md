EC2

- Là dịch vụ máy chủ ảo của AWS

- Khởi tạo EC2 sẽ sinh ra 1 primary ENI gắn vào EC2 đó. (Nếu stop EC2 → primary ENI cũng bị xóa theo). Primary ENI chứa các thông tin mạng:

  - Primary private IP (bắt buộc phải có tối thiểu 1 private IP). IP này sẽ show ra khi dùng lệnh “ip addr" và nó nằm ở trong VPC, chỉ truy cập nội bộ VPC.

  - Public IP (nếu chọn "Auto assign public IP" khi tạo EC2). IP này là IP NAT để map từ public IP vào primary private IP. IP này sẽ không show ra khi dùng lệnh "ip addr", phải curl về metadata để biết (http://169.254.169.254/latest/meta-data/public-ipv4)

  - Elastic IP (nếu gán)

  - SG

  - MAC Address

  Lưu ý: EC2 và ENI là AZ-level resources


- Ngoài primary ENI, EC2 có thể gán thêm các secondary ENI cho EC2. Số lượng ENI có thể gán thêm phụ thuộc vào loại EC2, ví dụ:

  - t2.micro: max 2 ENI

  - m5.large: max 3 ENI

- Mỗi secondary ENI sẽ tối thiểu có 1 private IP (giống như primary ENI). Khi dùng lệnh "ip add" thì sẽ show ra các interface (mỗi interface tương ứng với 1 ENI đã gán vào EC2) và private IP của từng ENI.

Ví dụ:
```
eth0: # primary ENI

inet 172.31.16.101/20 #primary private IP

inet 172.31.16.105/20 # secondary private IP

eth1: #secondary ENI

inet 172.31.16.102/20 #primary private IP
```

- Public IP có 2 loại là dynamic (tự sinh ra khi tạo EC2 với "auto assign public IP" và luôn map với primary private IP của primary ENI) và EIP (cố định - static, ít mất tiền hơn, sẽ có thể map vào primary private IP hoặc secondary private IP của bất kỳ ENI nào)

- Lưu ý:

  - EIP và private IP map với nhau theo mối quan hệ 1:1.
  - 1 EC2 có thể có nhiều public IP nếu gán EIP cho nhiều ENI
  - 1 ENI có thể có nhiều EIP nếu ENI đó có nhiều private IP.
  - Nếu EC2 có dynamic IP mà ta gán thêm EIP → dynamic IP sẽ biến mất.
