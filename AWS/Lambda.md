Giải thích thêm về handler: Handler là điểm vào (entry point) của Lambda function, được AWS Lambda gọi khi function được kích hoạt bởi một sự kiện (event).

Tại sao nên khởi tạo kết nối database bên ngoài handler?

Lambda Execution Context Reuse

- AWS Lambda giữ lại execution environment (bộ nhớ, biến global) sau khi function chạy xong để tái sử dụng cho các lần invoke tiếp theo (warm start).
- Code bên ngoài handler (ví dụ: khởi tạo DB connection) chỉ chạy một lần khi execution environment được khởi tạo.
- Code bên trong handler chạy mỗi lần invoke, dẫn đến việc kết nối DB được tạo mới liên tục → tăng latency và tốn tài nguyên.
