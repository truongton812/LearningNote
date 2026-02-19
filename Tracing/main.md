
Middleware (phần mềm trung gian) là một lớp phần mềm nằm giữa các thành phần khác nhau trong hệ thống, giúp chúng giao tiếp và xử lý dữ liệu với nhau một cách hiệu quả. Trong kiến trúc hệ thống phần mềm, middleware có thể xem là phần mềm nằm giữa hệ điều hành và các ứng dụng, hoặc giữa các dịch vụ với nhau, giúp cung cấp các dịch vụ chung như: API gateway, message broker, cache, session management, transaction manager…


Trong các framework web (Express, Laravel, ASP.NET, Flask…), middleware là các hàm/đoạn mã trung gian nằm giữa request và response. Middleware sẽ nhận request từ client, xử lý trước khi cho vào handler (handler là phần xử lý chính của một request) chính (controller, route handler…). Middleware giúp tách logic cross‑cutting (xác thực, log, bảo mật, cache…) ra khỏi logic kinh doanh, khiến code gọn, dễ bảo trì và tái sử dụng.

Mục đích của Middleware là để:
- Xác thực, phân quyền.
- Ghi log, bắt lỗi.
- Thêm/sửa header, nén dữ liệu, cache…

Sau khi xử lý, middleware có thể:
- Chuyển tiếp request sang middleware tiếp theo hoặc vào handler.
- Dừng ngay và trả về response ngay tại đó (ví dụ chặn truy cập không hợp lệ).


Trong lập tình, ta có thể thêm middleware vào pipeline request/response để đo/gán/thay đổi gì đó liên quan đến response, ví dụ như:
- Đo thời gian xử lý (thời gian từ lúc nhận request đến lúc trả response). Một middleware có thể ghi lại thời điểm bắt đầu và tính toán thời gian xử lý của request
- Ghi log thông tin response (status code, header, body…).
- Thêm/sửa header trên response (ví dụ: X-Response-Time, CORS, security headers…).
- Nén dữ liệu trước khi gửi đáp trả.
