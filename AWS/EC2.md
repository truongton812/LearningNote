# EC2

### Tổng quan về EC2
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

### EC2 pricing:
Có các loại:
- On-demand: standard, dùng bao nhiêu trả bấy nhiêu, không có discount/commitment
- Reserved: 1-3 năm commitment dùng EC2. Cần cam kết type (VD t2.micro), region, HĐH (window, linux)
  - Standard reserved instance: có thể thay đổi AZ, instance size, networking type
  - Convertable reserved instance: thay đổi được AZ, instance size, networking type, family, OS, tenancy và payment option
- Spot instance: EC2 có thể bị terminate bất kỳ lúc nào
  - spot instance: 1 hoặc nhiều spot EC2
  - spot fleet: duy trì số lượng spot/on-demand EC2 đáp ứng được dung lượng (capacity) cụ thể
  - EC2 fleet: duy trì số lượng nhất định spot/on-demand/reserved instance EC2
- Spot block: duy trì 1 số lượng EC2 nhất định trong 1 khoảng thời gian. Trong thời gian đó EC2 không bị terminate
- dedicated instance: máy chủ vật lý dành riêng cho 1 AWS account, tuy nhiên ta ko kiểm soát đc instance nào nằm trên máy chủ nào, isolate với các instance của khách hàng khác (vẫn cùng host) về mặt hardware. Trả tiền theo instance
- Dedicated host: máy chủ vật lý dành riêng cho 1 AWS account, ta có thể kiểm soát đc instance nào đặt trên máy chủ nào, isolate về mặt host. Biết được thông tin socket, core, hostID. Trả tiền theo host. Thích hợp cho các app cần licenses bound với server
- Saving plan: cam kết sử dụng EC2/ Fargate/ Lambda từ 1-3 năm. Ko cần cam kết instance type, region, HĐH. Giảm giá ít hơn reserved instance

### EC2 MIGRATION GIỮA CÁC AZ
Muốn move 1 EC2 từ AZ này sang AZ khác thì dùng AMI (AMI underlying sẽ tạo snapshot của EBS) (lưu ý chỉ move được cùng region)

### CROSS ACCOUNT AMI SHARING
Nếu muốn share AMI với account khác:

- AMI đấy có EBS là unencrypted snapshot -> share thoải mái

- AMI đấy có EBS là encrypted snapshot (lưu ý chỉ chấp nhận encrypted = KMS CMK) -> phải share cả customer managed key = IAM permission. Nếu ko account kia sẽ không giải mã được AMI dù được share

Cách share: vào AMI -> action -> Edit AMI permission

Lưu ý: Nếu muốn share 1 encrypted AMI cho account khác thì snapshot của AMI đấy phải unencrypt = KMS key; không thể share nếu snapshot đấy được encrypt bằng default AWS-managed key.

### CROSS ACCOUNT AMI COPY

Acc A share AMI cho acc B, acc B copy AMI đẩy vào acc B -> acc B là owner của copied AMI

Điều kiện: acc A phải grant read permission vào EBS cho acc B. Nếu AMI có EBS là encrypted snapshot thì acc A phải share cả key

Khi copy sang thì có thể encrypt bản copy = CMK của acc B

### EC2 Image Builder (Bổ sung sau)

### Instance profile

- Là phương tiện duy nhất để truyền IAM role cho EC2 instance. Khi muốn EC2 instance sử dụng quyền của 1 IAM role, ta cần tạo instance profile chứa role đó, sau đó gán instance profile cho EC2 instance.

- Khi EC2 instance được gán instance profile, AWS sẽ cung cấp cho instance đấy các temporary credential (thông qua instance metadata service tại địa chỉ http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>). Các SDK và CLI trên instance sẽ dùng credential này để truy cập tài nguyên AWS theo quyền của role.

- 1 instance profile chỉ chứa 1 IAM role, nhưng 1 IAM role có thể được gán vào nhiều instance profile khác nhau.

- Khi gán role cho EC2 trên AWS console thì instance profile sẽ được tạo tự động.

- Cách tạo instance profile bằng CLI:

  - Tạo IAM role: aws iam create-role ...

  - Gán policy cho role: aws iam attach-role-policy ...

  - Tạo instance profile: aws iam create-instance-profile

  - Thêm role vào instance profile: aws iam add-role-to-instance-profile

  - Gán instance profile cho EC2: aws ec2 associate-iam-instance-profile
