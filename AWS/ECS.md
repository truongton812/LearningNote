Amazon Elastic Container Service (ECS) là dịch vụ quản lý container của AWS giúp chạy và quản lý các ứng dụng container.
<img width="800" height="348" alt="image" src="https://github.com/user-attachments/assets/e84fb4d7-9296-404a-aa7c-a41531b1f09f" />

1. Task Definition (Định nghĩa tác vụ)
Đây là bản thiết kế (blueprint) của ứng dụng container bạn muốn chạy.

Task Definition mô tả chi tiết về một hoặc nhiều container (tối đa 10) gồm: image Docker dùng, CPU, bộ nhớ, cổng mở, volume dữ liệu, và cách khởi chạy.

Nó là một tập tin cấu hình (dạng JSON) quy định container nào và tham số thế nào.

Tạo task definition
```
{
  "family": "webserver",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "nginx",
      "memory": 100,
      "cpu": 99,
      "essential": true
    }
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "networkMode": "awsvpc",
  "memory": "512",
  "cpu": "256"
}
```

Ý nghĩa các phần

"family": Nhóm task definition; có thể sử dụng để versioning.

"containerDefinitions": Định nghĩa thông số từng container, bao gồm tên, image, dung lượng RAM, CPU.

"requiresCompatibilities": Chọn loại hạ tầng chạy container (ví dụ FARGATE, EC2, có thể chọn cả 2). -> khi triển khai task thì sẽ chọn để triển khai lên EC2 hay fargate hay cả 2

"networkMode": Chọn kiểu mạng (awsvpc dùng cho Fargate).

"memory" và "cpu": Dung lượng tổng cho Task definition.​

Bạn có thể custom thêm các trường như portMappings, environment, hoặc gắn volume theo nhu cầu

Trick: nếu tạo task definition bằng giao diện console mà chọn EC2/fargate mà sau đấy muốn update bằng giao diện console sẽ không được, nhưng nếu dùng "create new revision with json" để update thì sẽ được

2. Task (Tác vụ)
Một Task là một phiên bản chạy thực tế của một Task Definition.

Khi bạn khởi chạy một Task Definition trên cluster ECS, nó tạo ra một Task.

Task chạy các container theo định nghĩa trong Task Definition.

Các Task có thể chạy độc lập, hoặc do Service quản lý.


3. Service (Dịch vụ)
Service dùng để quản lý và duy trì số lượng Task chạy liên tục theo mong muốn.

Ví dụ, bạn có thể yêu cầu Service chạy luôn 3 bản sao của Task, ECS Service sẽ đảm bảo luôn có 3 Task chạy, tự động thay thế nếu Task nào đó ngừng.

Service giúp cân bằng tải, tích hợp với Load Balancer, và tự động mở rộng số Task.

Mối Quan Hệ Tóm Tắt:
Task Definition: Kế hoạch chi tiết.

Task: Phiên bản chạy thực tế.

Service: Đơn vị quản lý nhiều Task liên tục.


### Mối quan hệ

Khi vào ECS cluster sẽ có 2 tab ở dưới là service và task

Click vào để chọn tạo service hoặc task từ task definition

Trong mục tạo service có thể define số lượng task cần chạy, các task đấy sẽ thuộc sự quản lý của service 

Trong mục tạo task cũng có thể define số lượng task cần chạy, nhưng task run lên từ đấy không được ai quản lý cả


---

phân biệt task execution role và task role

Khi container khởi tạo lên sẽ có 2 giai đoạn:
- giai đoạn khởi động sẽ cần sử dụng task execution role (VD gọi ECR, viết log vào CW) -> task execution role là bắt buộc phải có, có thể dùng role do aws tạo sẵn tên là AmazonECSTaskExecutionRolePolicy
- giai đoạn thực thi sau khi khởi động lên sẽ dùng task role (VD gọi vào dynamodb, s3,..)

---

Sự khác biệt giữa Farget và EC2 launch type

Khi tạo EC2 launch type thì các task có thể nằm chung trong 1 EC2 instance, muốn truy cập phải thông qua IP của EC2 instance + port được cấp ngẫu nhiên

Khi tạo Fargate thì mỗi task sẽ có 1 IP riêng (cả IP public và IP private), truy cập thông qua IP của task, và port là port của container chứ ko phải ngẫu nhiên. Câu hỏi: Nếu hết ip private thì có sinh ra thêm được task mới không? Trả lời là không. Fargate thì không thể xem log = lệnh docker logs được, chỉ có thể xem trên giao diện

---

Các loại network trong ECS

- host mode: map giữa ip của host và eni của container??? không thể run nhiều replicas của 1 task do sẽ bị conflict port
- bridge mode: map giữa ip của host và eni của container, sau đó đẩy đến network bridge??? Có thể run nhiều replicas của 1 task. Nhược điểm là dynamic port
- awsvpc mode

Amazon ECS hỗ trợ 4 chế độ network (network mode) khác nhau để kiểm soát cách container giao tiếp với nhau, với host, và mạng bên ngoài.​

Khi dùng fargate thì chỉ  dùng được awsvpc

Khi dùng ec2 có thể dùng awsvpc và bridge


1. Bridge mode (Mặc định trên Linux)
Dựa trên cơ chế Docker bridge network.

Mỗi container trong cùng một host sẽ kết nối qua một mạng ảo nội bộ (docker0 bridge).

Cho phép port mapping động (ví dụ: container chạy port 80 nhưng ánh xạ thành port ngẫu nhiên trên host).

Ưu điểm: dễ cấu hình, cách biệt giữa các container.

Nhược điểm: có overhead hiệu năng do xử lý NAT nội bộ.

2. Host mode
Container dùng trực tiếp network interface của host EC2.

Container chia sẻ IP và network stack với EC2 instance chứa nó.

Ưu điểm: hiệu năng cao hơn Bridge (do bỏ qua lớp NAT).

Nhược điểm: không thể chạy nhiều container cùng port trên cùng một host, vì trùng cổng sẽ xung đột.

3. awsvpc mode
Mỗi Task có Elastic Network Interface (ENI) riêng trong VPC.

Mỗi task có địa chỉ IP riêng như một EC2 instance độc lập và có thể gán Security Group riêng.​

Đây là chế độ duy nhất được ECS Fargate hỗ trợ.

Ưu điểm: cách ly mạnh, quản lý bảo mật và kết nối chi tiết hơn.

Nhược điểm: chiếm tài nguyên nhiều hơn (mỗi task thêm 1 ENI).

4. None mode
Tắt hoàn toàn mạng bên ngoài container.

Chỉ tồn tại loopback interface bên trong container.

Không hỗ trợ port mapping.

Dùng cho trường hợp đặc biệt (container không cần giao tiếp ngoài, hoặc tự định nghĩa driver mạng riêng).

| Network mode | IP riêng cho task        | Cần EC2 Host    | Hỗ trợ Fargate | Hiệu năng         | Port mapping |
|--------------|--------------------------|-----------------|----------------|-------------------|--------------|
| bridge       | Không                    | Có              | Không          | Trung bình        | Có           |
| host         | Không (chia sẻ host)     | Có              | Không          | Cao               | Không        |
| awsvpc       | Có                       | Có hoặc Fargate  | Có             | Cao               | Có           |
| none         | Không                    | Có              | Không          | Không giao tiếp   | Không        |


Trong thực tế:

Fargate chỉ cho phép awsvpc.

Với EC2 launch type, bạn có thể chọn giữa bridge, host, hoặc awsvpc tùy yêu cầu hiệu năng, bảo mật và khả năng mở rộng.​

---

Khi nào nên dùng ec2 / fargate

Fargate: tối đa là 16 vcpu và 120gb ram. EC2 có thể dùng theo nhu cầu

EC2 có thể custom về GPU (fargate không có), EBS (nhu cầu IOPS cao)

---

ELB và ECS

Ta có thể đặt task ở public subnet, lúc đấy task sẽ có public IP và user có thể truy xuất  task thông qua public IP. Ta có thể dùng ELB để đẩy traffic thay vì phải nhớ từng ip của task

<img width="652" height="550" alt="image" src="https://github.com/user-attachments/assets/c5a70e40-1422-40f9-a06d-5bba0af66ef4" />


Tuy nhiên đấy không phải best practice về bảo mật

Best practice là đặt task ở private subnet và dùng 1 ELB để đẩy traffic đến task

Khi tạo ELB cho ECS fargate thì phải tạo target group type là IP (do chỉ có container IP) thì sau này ở Service mới nhận

Hoặc để đơn giản thì khi tạo load balancer cứ tạo bừa 1 target group, sau này trong lúc tạo service thì tạo lại target group mới

---

ECS Autoscaling

Giúp scale task dựa trên working

Khi dùng target tracking policy thì có 3 metric cơ bản là CPU, RAM và số request. Có thể tạo custom metric (tham khảo thêm)

---

ECS metric

Nếu enable Container insight thì có thêm nhiều metric hơn (mất phí)

---

ECS log

Để gửi log vào Cloudwatch log thì:
- Nếu dùng fargate thì chỉ cấn gán ecstaskexecutionrole cho task
- Nếu dùng EC2 thì phải cài cloudwatch agent và gán role cho EC2 (cần 2 quyền là ECSContainerServiceforEC2Role và SSMManagedInstanceCore)  để có quyền push log lên cloudwatch

---

Capacity providers quản lý compute infrastructure cho cluster ECS — ví dụ EC2 Auto Scaling Group hoặc Fargate.​

Một ECS cluster có thể gán nhiều capacity provider gồm cả EC2 và Fargate.​

Khi tạo ECS Service, bạn được chọn 1 hoặc nhiều capacity provider trong phần strategy, đặt weight và base cho từng loại.​

Thường dùng để chia workload giữa nhiều nhóm EC2 (khác cấu hình, khác pricing model, v.v.) hoặc giữa Fargate và Fargate Spot.​

Quan trọng: Mỗi capacity provider strategy thuộc một service chỉ dùng được cùng loại provider — hoặc toàn bộ là các EC2 Auto Scaling Group, hoặc toàn bộ là các Fargate (Fargate/Fargate Spot). Không thể pha trộn EC2 và Fargate cùng lúc cho strategy của một
