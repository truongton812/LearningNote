###### không phải lúc nào Helm chart cũng hỗ trợ trực tiếp mọi cấu hình mà người dùng cần thông qua biến values.yaml. Khi bị hạn chế bởi biểu mẫu có sẵn của chart mà không thể tinh chỉnh một trường nào đó, Kustomize sẽ được dùng để "đắp vá" (patch) lại manifest đã được sinh ra từ Helm chart, giúp tùy biến những chỗ chart không hỗ trợ.


Helm chart chỉ cho phép chỉnh sửa các trường đã được người viết chart tạo biến hóa sẵn trong values.yaml.

Nhiều trường hợp chart của bên thứ ba (ví dụ như chart NGINX, Prometheus, Istio...) không có biến để cấu hình chi tiết những gì mình cần – như thêm priorityClassName, chỉnh type của Service v.v.

Fork chart để sửa sẽ gặp rắc rối lớn khi upstream cập nhật phiên bản, mã nguồn bị phân nhánh và khó bảo trì.

Giải pháp: Patch bằng Kustomize

Render manifest từ Helm (bằng lệnh helm template hoặc dùng post-renderer của Helm).

Dùng Kustomize để patch những trường hoặc resource mà chart không hỗ trợ bằng cách overlay và patch trực tiếp vào YAML kết quả.

Lối đi này giúp tận dụng được điểm mạnh của cả Helm (quản lý gói) lẫn Kustomize (patch và tùy biến YAML linh hoạt).

###### Kustomize không hỗ trợ thay biến kiểu ${MY_VAR} ngay tại runtime mà cần phải build manifest trước rồi mới apply
