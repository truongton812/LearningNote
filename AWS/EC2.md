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

- EIP là regional resource, có thể move giữa các ENI. Khi gán EIP vào ENI, thực chất đã tạo 1 NAT giữa EIP và private IP (primary hoặc secondary) của ENI → truy cập public IP thì kết nối đến private IP (NAT 1-1).

- Private IP là IP nằm bên trong VPC, chỉ truy cập nội bộ, từ internet cần NAT.

- Các loại Network interfaces:

  - ENI: là network interface chuẩn, cơ bản nhất, dùng với tất cả các loại EC2.
  - ENA (elastic network adapter): là phiên bản nâng cao, cung cấp enhanced networking với hiệu năng cao, dùng cho các tác vụ cần băng thông lớn và độ trễ thấp. Chỉ hỗ trợ 1 vài loại EC2.
  - EFA (Elastic fabric adapter): là network adapter chuyên dùng cho High Performance Computing (HPC) và Machine Learning với các yêu cầu latency thấp và throughput cao, dùng với tất cả các loại EC2.

### EC2 health check:

Là built-in function của EC2 (check ở tab "Status and alarm" trong EC2 console), giúp check 3 loại:

- System status check: check về mặt vật lý (nguồn, mạng, phần cứng) thực hiện bởi lớp vật lý. Nếu có lỗi thì ta stop/start lại → EC2 sẽ move sang host khác (có public IP mới).

- Instance status check: check về lớp OS (CPU, ram, OS, driver), thực hiện bởi chính OS và lớp OS của hóa.

- Attached EBS status check: check trạng thái EBS của EC2.

EC2 health check có thể dùng cho:

- ASG (mặc định ASG dùng health check này).

- ELB (optional, do ELB cũng có health check riêng).
