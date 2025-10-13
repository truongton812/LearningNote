
Option "Availability Zones and subnets" khi tạo Application Load Balancer trên AWS cho phép chọn khu vực khả dụng (AZ - Availability Zone) và các subnet tương ứng mà Load Balancer sẽ hoạt động trong đó và phân phối traffic tới các EC2 instance hoặc resources bên trong các subnet này

Chỉ những targets (EC2 instance, ECS,… ) nằm trong các subnet được chọn thì mới nhận traffic từ load balancer.

mặc dù về mặt kiến trúc, một AZ có thể có nhiều subnet, thì trong quá trình tạo và cấu hình ALB, bạn chỉ được phép chọn một subnet cho mỗi AZ để ALB tạo ra các node load balancer trong đó

Khi tạo Application Load Balancer (ALB) trên AWS, AWS sẽ tạo một hoặc nhiều node load balancer ở mỗi Availability Zone (AZ) bạn chọn. Mỗi node này được đại diện bằng một Elastic Network Interface (ENI) trong subnet mà bạn đã chọn ở AZ đó.

Elastic Network Interface (ENI) là một thành phần mạng logic trong VPC, tương tự như một card mạng ảo (virtual NIC). ENI là một interface mạng logic được gán vào một subnet cụ thể trong VPC.


Khi bạn chọn một subnet trong một AZ để tạo ALB, AWS sẽ tạo một ENI trong subnet đó. ENI này có địa chỉ IP trong dải subnet và mang nhiệm vụ nhận và truyền tất cả lưu lượng mạng (traffic) đến/đi từ nod load balancer trong AZ đó.

Mỗi node load balancer tương ứng với một ENI riêng biệt trong subnet đã chọn, làm điểm xử lý mạng cho ALB trong vùng AZ đó.

ENI này giữ vai trò như gateway nhận request từ client và chuyển tiếp request đến các target (ví dụ EC2 instance) trong target group.

AWS cần quản lý ENI này riêng để có thể bảo trì, tự động thay thế khi cần, cũng như đảm bảo đủ IP để mở rộng (scale) load balancer.

---

Security Group (SG) cho Application Load Balancer (ALB) là một thành phần tường lửa ảo giúp kiểm soát lưu lượng mạng vào và ra của chính ALB.

SG của ALB kiểm soát các kết nối đến từ client (người dùng hoặc dịch vụ ngoài).

Ví dụ, SG của ALB thường mở các port phổ biến như HTTP (80), HTTPS (443) để nhận lưu lượng truy cập web.

ALB sẽ chấp nhận hoặc từ chối các yêu cầu dựa trên quy tắc SG này.

Mối liên hệ giữa Security Group của ALB và Security Group của EC2

EC2 instance thường có SG riêng riêng biệt, kiểm soát lưu lượng Vào/Ra từ instance đó.

Khi ALB chuyển tiếp traffic đến các EC2 (thường qua Target Group), các request này được thực hiện giữa ALB và các EC2 trong mạng nội bộ. Lưu ý cần cấu hình Security Group cho EC2 để chấp nhận traffic từ ALB, chứ Không cấu hình cho target group, vì target group chỉ là tập hợp các target (EC2), không phải entity có SG riêng.

Do đó, để EC2 nhận traffic từ ALB, SG của EC2 phải cho phép nhận lưu lượng từ SG của ALB (chấp nhận inbound traffic từ SG của ALB).

Nói cách khác, SG của EC2 cần có rule inbound cho phép traffic đến từ SG của ALB, thường là cho các port ứng dụng mà EC2 đang chạy (ví dụ port 80 cho web).

