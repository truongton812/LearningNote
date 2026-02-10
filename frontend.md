#### Ứng dụng client-side
Ứng dụng client‑side (kiểu SPA React/Vue/Angular) về bản chất là: server chỉ đưa cho browser “bộ code”, còn mọi thứ chính diễn ra trong trình duyệt.

Chu trình từ lúc user gõ URL
- User gõ https://my-frontend.com vào trình duyệt.
- Browser gửi HTTP(S) request đến server (S3, EC2, Nginx, v.v.).
- Server trả về một file HTML “khung” (thường rất đơn giản) kèm các file JS/CSS tĩnh (bundle React/Vue như main.js, bundle.js, CSS, image…) -> Đến đây server coi như hết việc “render giao diện”; phần còn lại là do file JS (do server gửi xuống) trên client lo.
- File JS sẽ có nhiệm vụ render giao diện (tạo HTML, thay đổi DOM, chuyển route SPA), xử lý sự kiện (click, input, submit form…), gọi API (axios, fetch) đến backend (ALB, API Gateway, ECS, EC2…). Mọi việc chuyển trang, đổi state, update UI đều xử lý trong browser.
- Nếu trong code frontend có gọi đến backend thì browser lúc này tự tạo HTTP request đến backend, request này đi trực tiếp từ máy người dùng → ALB / API Gateway / backend.
- Trong mô hình client-side, dù frontend được host ở S3, EC2, hay CloudFront thì frontend server chỉ serve file tĩnh, không call API xuống backend mà thực chất là browser của người dùng call API xuống. Do đó backend chỉ thấy IP của client, không thấy “IP của S3” hay “IP của EC2” -> không thể thực hiện whitelist frontend theo IP được

#### Ứng dụng server‑side?
Server‑side: mỗi lần user vào trang, browser gửi request tới server → server xử lý, render HTML hoàn chỉnh (đã có data) → trả về cho browser.
