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

"requiresCompatibilities": Chọn loại hạ tầng chạy container (ví dụ FARGATE, EC2).

"networkMode": Chọn kiểu mạng (awsvpc dùng cho Fargate).

"memory" và "cpu": Dung lượng tổng cho Task definition.​

Bạn có thể custom thêm các trường như portMappings, environment, hoặc gắn volume theo nhu cầu


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
