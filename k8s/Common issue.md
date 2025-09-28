###### không phải lúc nào Helm chart cũng hỗ trợ trực tiếp mọi cấu hình mà người dùng cần thông qua biến values.yaml. Khi bị hạn chế bởi biểu mẫu có sẵn của chart mà không thể tinh chỉnh một trường nào đó, Kustomize sẽ được dùng để "đắp vá" (patch) lại manifest đã được sinh ra từ Helm chart, giúp tùy biến những chỗ chart không hỗ trợ.


Helm chart chỉ cho phép chỉnh sửa các trường đã được người viết chart tạo biến hóa sẵn trong values.yaml.

Nhiều trường hợp chart của bên thứ ba (ví dụ như chart NGINX, Prometheus, Istio...) không có biến để cấu hình chi tiết những gì mình cần – như thêm priorityClassName, chỉnh type của Service v.v.

Fork chart để sửa sẽ gặp rắc rối lớn khi upstream cập nhật phiên bản, mã nguồn bị phân nhánh và khó bảo trì.

Giải pháp: Patch bằng Kustomize

Render manifest từ Helm (bằng lệnh helm template hoặc dùng post-renderer của Helm).

Dùng Kustomize để patch những trường hoặc resource mà chart không hỗ trợ bằng cách overlay và patch trực tiếp vào YAML kết quả.

Lối đi này giúp tận dụng được điểm mạnh của cả Helm (quản lý gói) lẫn Kustomize (patch và tùy biến YAML linh hoạt).

###### Kustomize không hỗ trợ thay biến kiểu ${MY_VAR} ngay tại runtime mà cần phải build manifest trước rồi mới apply

CI/CD pipeline muốn truyền biến image tag vào manifest khi deploy. Helm cho phép truyền biến này dễ dàng, còn Kustomize phải dùng workaround như build trước rồi apply hoặc sử dụng thêm công cụ phụ trợ như envsubst – khá bất tiện và đôi khi gây lỗi.

Muốn scale số lượng replica theo biến môi trường build, Kustomize không có giải pháp runtime thực sự, còn Helm lại đang là lựa chọn phổ biến để giải quyết vấn đề này.

"Truyền biến động ở runtime" trong ngữ cảnh triển khai Kubernetes có nghĩa là truyền hoặc thay thế giá trị biến vào manifest tại thời điểm triển khai, chứ không phải lúc tạo hoặc lưu file manifest trước đó. Tức là, giá trị cụ thể (ví dụ: image tag, tên môi trường, số replica) sẽ được chỉ định và chèn trực tiếp vào YAML ngay khi chạy lệnh deploy, tùy thuộc vào hoàn cảnh thực tế hoặc kết quả CI/CD pipeline.

Ý nghĩa "runtime" ở đây
"Runtime" nghĩa là vào lúc thực thi, khi deployment đang được thực hiện trên môi trường thật (ví dụ: production, staging), chứ không phải chuẩn bị file manifest từ trước.

Ví dụ, khi CI/CD pipeline build xong image v1.2.3, nó sẽ truyền tag này vào manifest thông qua biến runtime rồi triển khai luôn lên cluster. Nếu dùng Helm, chỉ cần chạy helm install --set image.tag=v1.2.3 là biến được truyền vào và manifest sẽ sinh ra đúng giá trị tag đó.

Kustomize thì không hỗ trợ truyền biến ở thời điểm này một cách native
Không thể trực tiếp sửa giá trị trong file patch.yaml của Kustomize để truyền biến động ở runtime theo cách native của Kustomize. File patch.yaml là tĩnh, tức nó chứa giá trị cố định, được áp dụng lúc build manifest trước khi deploy, không hỗ trợ thay biến hoặc nhận biến từ môi trường runtime như Helm. nếu muốn thay đổi phải build lại file trước hoặc dùng công cụ ngoài như dùng script hoặc công cụ bên ngoài như envsubst, sed, hoặc CI/CD pipeline để thay giá trị biến trong patch.yaml trước khi chạy Kustomize build hoặc deploy

Lý do
Kustomize chỉ xử lý các file YAML một cách tĩnh, không có cơ chế native cho phép chèn biến runtime trực tiếp vào patch.yaml khi apply manifest lên cluster.

Biến động muốn truyền phải được chuẩn bị trước bằng cách chỉnh sửa patch.yaml hoặc các file overlay trước khi chạy kubectl apply hoặc kustomize build, tức là lúc build manifest, không phải lúc runtime thực tế deploy.

Minh họa thực tế
Truyền biến động ở runtime hay dùng để:

Gán tag image mới build được từ CI/CD.

Truyền cấu hình môi trường (env, replica, ingress host…) theo môi trường deploy hôm đó.

Update secret/token hoặc cấu hình nhạy cảm mà phải lấy từ CI/CD hoặc vault
