1. --advertise-client-urls
Ý nghĩa:
Địa chỉ (URL) mà etcd sẽ thông báo cho các client (ví dụ: kube-apiserver, các công cụ quản trị, hoặc các node khác) để kết nối đến dịch vụ etcd này.

Vai trò:
Thông báo cho các thành phần khác biết địa chỉ truy cập etcd node này để thực hiện các thao tác đọc/ghi dữ liệu.

Ví dụ:
--advertise-client-urls=https://192.168.1.10:2379

2. --initial-advertise-peer-urls
Ý nghĩa:
Địa chỉ (URL) mà etcd sẽ dùng để thông báo với các thành viên khác trong cụm etcd về endpoint peer của chính nó.

Vai trò:
Các node etcd khác sẽ sử dụng địa chỉ này để kết nối peer-to-peer với node này trong quá trình khởi tạo hoặc khi cluster hoạt động.

Ví dụ:
--initial-advertise-peer-urls=https://192.168.1.10:2380

3. --listen-client-urls
Ý nghĩa:
Danh sách các địa chỉ (URL) mà etcd sẽ lắng nghe (listen) để nhận kết nối từ client (kube-apiserver, etc.).

Vai trò:
Xác định các interface và port mà etcd sẽ mở ra để phục vụ các request từ client.

Ví dụ:
--listen-client-urls=https://127.0.0.1:2379,https://192.168.1.10:2379

4. --listen-peer-urls
Ý nghĩa:
Danh sách các địa chỉ (URL) mà etcd sẽ lắng nghe để nhận kết nối từ các peer (các thành viên khác trong cụm etcd).

Vai trò:
Xác định interface và port dùng cho giao tiếp nội bộ giữa các node etcd.

Ví dụ:
--listen-peer-urls=https://192.168.1.10:2380

5. --initial-cluster
Ý nghĩa:
Danh sách các thành viên ban đầu của cụm etcd, định dạng name=peerURL, phân tách bằng dấu phẩy.

Vai trò:
Giúp các node etcd biết được các peer ban đầu để hình thành cluster, đồng bộ và giao tiếp với nhau.

Ví dụ:
--initial-cluster=master1=https://192.168.1.10:2380,master2=https://192.168.1.11:2380,master3=https://192.168.1.12:2380

Tóm tắt:

Các trường advertise dùng để thông báo địa chỉ cho các thành phần khác biết cách kết nối đến node etcd này.

Các trường listen xác định etcd sẽ mở các cổng nào để nhận kết nối.

initial-cluster giúp các node etcd nhận diện nhau và hình thành cluster phân tán.
