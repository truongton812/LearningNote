# Cách tổ chức và quản lý tài khoản AWS


Theo gợi ý của AWS thì đối với các doanh nghiệp có quy mô lớn thì ta nên triển khai hệ thống theo kiểu Landing Zone với nhiều tài khoản AWS khác nhau. Mục đích của việc tạo nhiều tài khoản là để tránh bị giới hạn AWS Service Quotas, tránh bị ảnh hưởng toàn bộ nếu có một tài khoản nào đó bị hack và dễ dàng quản lý AWS Billing.


Tiêu chí phổ biến để tạo tài khoản là dựa theo môi trường và những thành phần phổ biến trong một hệ thống phần mền.

Ví dụ khi phát triển sản phẩm thì ta có môi trường là DEV, UAT, STAGING, PROD. Đối với các môi trường như DEV, UAT, STAGING có thể gôm lại thành môi trường nonprod. Còn những thành phần phổ biến trong một hệ thống thì bao gồm Networking, Workload, Operation (CI/CD), Monitoring, Logging, Data System.


Vậy ta có thể tạo các tài khoản với tên như sau (nonprod dùng cho các môi trường không phải production):
```
networking-nonprod
workload-nonprod
operation-nonprod
observability-nonprod
data-nonprod
networking-prod
workload-prod
operation-prod
observability-prod
data-prod
```

Trong đó:
- Networking dùng để quản lý network ra vào, tất cả các request đi vào và ra đều phải đi qua networking account trước rồi mới đi tiếp. Mục đích là để ta truy vết được toàn bộ request đi vào ra hệ thống.
- Workload account dùng để triển khai ứng dụng, database, cache.
- Operation account dùng để thực thi các tác vụ liên quan tới CI/CD, tạo hạ tầng cho các tài khoản khác.
- Observability account dùng để triển khai hệ thống monitoring, logging và tracing.
- Data account dùng để thực thi các tác vụ về dữ liệu như thu thập, xử lý dữ liệu và hiển thị dữ liệu đẹp cho người dùng nội bộ.


Khi tạo nhiều tài khoản như trên thì ta quản lý như thế nào về mặt AWS Billing và truy cập. Ta cần một một tài khoản nữa với mục đích là để quản lý toàn bộ tài khoản trên, có thể đặt tên tài khoản này là root. Để tài khoản root có thể quản lý được các tài khoản trên thì ta cần sử dụng AWS Organizations kết hợp với AWS Control Tower.

Vấn đề tiếp theo là về việc truy cập các tài khoản khác nhau. Khi ta có nhiều tài khoản AWS như trên thì khi truy cập ta cần làm thế nào để tiện nhất? Ta không thể dùng Console thông thường rồi login và logut để truy cập từng tài khoản được. Bên cạnh đó còn vấn đề về IAM User và Premission, nếu có một bạn cần truy cập nhiều tài khoản thì không lẻ ta phải vào từng tài khoản để tạo IAM User cho bạn? Để dễ dàng truy cập các tài khoản trong Control Tower thì AWS hỗ trợ dịch vụ IAM Identity Center. Ta chỉ cần tạo quyền và user ở một nơi và họ có thể truy cập được các tài khoản khác nhau thông qua IAM Identity Center, hình minh họa.
