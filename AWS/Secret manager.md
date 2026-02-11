AWS Secrets Manager tính phí theo 2 thành phần chính, không tính theo “một lần” mà theo tháng và theo số lần gọi API.

1. Chi phí cơ bản: Storage / mỗi secret

$0.40 mỗi secret mỗi tháng (khoảng $0.40/secret/tháng), tính theo giờ (pro‑rated).

Ví dụ: 10 secret → khoảng $4.00/tháng.

Không bị charge nếu secret đã bị xóa (hoặc đang trong recovery window).

2. API calls

$0.05 / 10,000 API calls (gồm GetSecretValue, PutSecretValue, DeleteSecret, RotateSecret, v.v.).

Ví dụ: 50,000 calls/tháng → khoảng $0.25,


3. Lưu ý quan trọng

Mỗi secret replica trong multi‑region được tính là 1 secret riêng → $0.40 cho mỗi replica.

Không bị charge cho việc tạo nhiều phiên bản (secret version), chỉ tính 1 lần cho secret đang tồn tại.

Các secret nhỏ (dưới 64KB, AWS khuyến nghị ≤10KB) không bị thêm phí lưu trữ.

→ Tổng khoảng $0.65–$1.00/tháng trong trường hợp dùng vừa phải.

