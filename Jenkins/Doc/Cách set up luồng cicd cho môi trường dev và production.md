### Cách set up luồng cicd cho môi trường dev và production


Khi thiết lập luồng CI/CD trên AWS CodePipeline cho môi trường dev và production, bạn có thể chọn tạo 1 pipeline duy nhất hoặc 2 pipeline riêng biệt. Dưới đây là ưu nhược điểm của từng cách để bạn cân nhắc:

1. Tạo 1 Pipeline Cho Cả Dev và Production
Ưu điểm
Quản lý đơn giản: Chỉ cần theo dõi và bảo trì một pipeline duy nhất.

Tự động liền mạch: Quá trình từ dev đến production được kết nối liên tục trong cùng pipeline, phù hợp với workflow có bước approval giữa dev và prod.

Tiết kiệm chi phí: Ít resource quản lý hơn, giảm phí dịch vụ (một số trường hợp, số lượng pipeline có thể ảnh hưởng đến chi phí).

Nhược điểm
Khó kiểm soát khi lỗi: Nếu pipeline bị lỗi ở môi trường dev sẽ ảnh hưởng đến toàn pipeline.

Phức tạp trong cấu hình: Cần thêm bước quản lý điều kiện (conditional actions) để xử lý deployment riêng cho dev và prod trong cùng pipeline.

Khó mở rộng: Rất lớn hoặc phức tạp khi thêm nhiều môi trường khác (staging, testing...).

2. Tạo 2 Pipeline Riêng Biệt Cho Dev và Production
Ưu điểm
Tách biệt rõ ràng: Mỗi pipeline phục vụ riêng một môi trường, dễ theo dõi, vận hành và xử lý sự cố riêng biệt.

An toàn hơn: Lỗi ở pipeline dev không ảnh hưởng đến pipeline production.

Dễ dàng mở rộng: Khi có nhiều môi trường thì pipeline cho từng môi trường rất linh hoạt.

Kiểm soát quyền riêng biệt: Dễ quản lý phân quyền truy cập pipeline và thông tin nhạy cảm theo môi trường.

Nhược điểm
Quản lý phức tạp hơn: Cần duy trì nhiều pipeline, dễ bị rối khi có nhiều pipeline.

Chi phí có thể cao hơn: AWS tính phí dựa trên số lượng pipeline và resource sử dụng.

Cần phối hợp giữa các pipeline: Ví dụ pipeline production phải chờ pipeline dev hoàn thành và trigger thủ công hoặc tự động.



##### Cách triển khai 1 pipeline

Tóm tắt luồng workflow:

Developer push/merge vào dev → trigger pipeline để build-test-deploy lên dev env → QA kiểm thử.

Khi có sự kiện merge vào main (hoặc đánh tag) → trigger pipeline để build-test (có thể deploy lên dev env) → yêu cầu manual approval → deploy lên production.

![image info](1.png)

Để xử lý luồng trên có thể tạo Pipeline đa nhánh (multibranch pipeline) hoặc pipeline có điều kiện (conditional stages) 

### Hệ thống CI/CD kết hợp với GitOps sử dụng ArgoCD

một hệ thống CI/CD kết hợp với GitOps sử dụng ArgoCD, quy trình hoạt động thực tế sẽ như sau:

1. Phát triển ứng dụng & quản lý source code

Nhà phát triển thực hiện các thay đổi trong mã nguồn và đẩy code lên repo quản lý source code (thường là trên GitHub hoặc GitLab). Đây là nơi duy nhất được tin cậy để lưu source code gốc và tài liệu liên quan.

2. Pipeline CI tự động build & kiểm thử

Ngay khi code được push lên, pipeline CI (Jenkins, GitHub Actions…) sẽ tự động được kích hoạt để:

Build project, kiểm thử chất lượng mã nguồn.

Tạo ra artifact (ví dụ: Docker image).

Scan bảo mật, kiểm thử tích hợp.

Đẩy hình ảnh lên image registry (Docker Hub, ECR, …).

3. Quản lý cấu hình triển khai qua GitOps repo

Sau khi build thành công, pipeline sẽ cập nhật tag phiên bản image mới vào các file manifest (YAML hoặc Helm/Kustomize) nằm trong repository quản trị triển khai (GitOps repo). Các thay đổi này được quản lý chặt chẽ bằng Git—mỗi commit đều mang ý nghĩa lịch sử triển khai.

4. Kiểm duyệt & duyệt triển khai

Pipeline có thể dừng lại ở đây để chờ quá trình approve (duyệt), đảm bảo chỉ có artifact an toàn, đã kiểm thử mới được deploy. Sau khi approve, pipeline sẽ thực hiện commit manifest mới vào repo GitOps.

5. ArgoCD tự động đồng bộ lên Kubernetes cluster

ArgoCD hoạt động như một agent, liên tục theo dõi repo GitOps. Nếu phát hiện commit mới (ví dụ tag image mới, hoặc chỉnh sửa cấu hình), nó sẽ đồng bộ hóa trạng thái thực tế của Kubernetes cluster về đúng như cấu hình miêu tả trên repo:

Tạo thêm, sửa đổi hoặc xóa tài nguyên (pod, service, configmap…) theo manifest.

Quá trình này hoàn toàn tự động, không cần thao tác tay ngoài commit vào Git repo.

Toàn bộ lịch sử thay đổi đều có log rõ ràng giúp audit và rollback rất dễ dàng.
