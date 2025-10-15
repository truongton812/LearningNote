Amazon Elastic Container Service (ECS) là dịch vụ quản lý container của AWS giúp chạy và quản lý các ứng dụng container.

1. Task Definition (Định nghĩa tác vụ)
Đây là bản thiết kế (blueprint) của ứng dụng container bạn muốn chạy.

Task Definition mô tả chi tiết về một hoặc nhiều container (tối đa 10) gồm: image Docker dùng, CPU, bộ nhớ, cổng mở, volume dữ liệu, và cách khởi chạy.

Nó là một tập tin cấu hình (dạng JSON) quy định container nào và tham số thế nào.

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
