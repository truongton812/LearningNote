CNI (Container Network Interface) là một đặc tả (specification) và bộ thư viện dùng để cấu hình network interfaces cho các container trong Linux.

CNI định nghĩa một giao diện chuẩn giữa container runtime (như Kubernetes, Podman) và các network plugin, giúp chúng hoạt động với nhau mà không cần phụ thuộc vào nhau.

Các thành phần chính của CNI
1. Specification (Đặc tả): Định nghĩa cách một plugin nhận input và trả output — thông qua biến môi trường và stdin/stdout theo định dạng JSON.
2. Plugin: Là các file thực thi (executables) thực hiện việc cấu hình mạng. Có hai loại:
- Interface plugin: tạo network interface (ví dụ: bridge, macvlan, ipvlan)
- Chained plugin: bổ sung thêm tính năng (ví dụ: portmap, bandwidth, firewall)

3. Library (libcni): Thư viện Go giúp tích hợp CNI vào container runtime dễ dàng hơn.


