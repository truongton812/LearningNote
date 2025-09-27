hoàn toàn có thể thêm VMnet2 (và các VMnet khác) để chạy song song với VMnet0. Đây là một tính năng rất mạnh mẽ của VMware Workstation, cho phép bạn tạo ra các mô hình mạng phức tạp.

Việc VMnet0 đang được sử dụng không ngăn cản bạn tạo và sử dụng các mạng ảo khác. Mỗi VMnet hoạt động như một "switch" ảo độc lập. Bạn có thể tạo nhiều switch ảo (VMnet2, VMnet3,...) và kết nối các máy ảo của mình vào chúng theo các cách khác nhau.

##### Cách thêm và cấu hình VMnet2

Bạn sẽ sử dụng công cụ Virtual Network Editor để thực hiện việc này.

Mở Virtual Network Editor:

Trong VMware Workstation, vào menu Edit > Virtual Network Editor.

Bạn có thể sẽ cần quyền Administrator. Nếu thấy nút Change Settings, hãy nhấp vào đó và xác nhận UAC (User Account Control).

Thêm Mạng Mới (Add Network):

Nhấp vào nút Add Network.

Một cửa sổ sẽ hiện ra, cho phép bạn chọn tên cho mạng mới (ví dụ VMnet2). Chọn VMnet2 từ danh sách và nhấn OK.

Cấu hình cho VMnet2:

Sau khi thêm, VMnet2 sẽ xuất hiện trong danh sách. Bây giờ bạn cần quyết định VMnet2 sẽ hoạt động ở chế độ nào:

Bridged: Nếu bạn muốn VMnet2 cũng là một mạng Bridged, hãy chọn chế độ Bridged và trong phần "Bridged to", bạn có thể chọn một card mạng vật lý khác trên máy thật (nếu có, ví dụ một card WiFi trong khi card Ethernet đang dùng cho VMnet0).

Host-only: Đây là lựa chọn phổ biến để tạo một mạng riêng tư, bị cô lập, chỉ cho phép giao tiếp giữa máy thật và các máy ảo được kết nối vào VMnet2.

NAT: Tương tự, bạn cũng có thể cấu hình VMnet2 như một mạng NAT riêng biệt với VMnet8.

Thiết lập dải IP (Nếu là Host-only hoặc NAT):

Nếu bạn chọn Host-only hoặc NAT, bạn có thể tùy chỉnh dải địa chỉ IP cho mạng VMnet2 trong phần Subnet IP. Ví dụ: 192.168.200.0.

Bạn cũng có thể cấu hình dịch vụ DHCP cho mạng này nếu muốn các máy ảo tự động nhận IP.

Áp dụng thay đổi: Nhấn Apply và sau đó OK để lưu cấu hình.

##### Cách sử dụng VMnet2 cho máy ảo

Sau khi đã tạo VMnet2, bạn có thể kết nối máy ảo của mình vào mạng này:

Thay đổi card mạng hiện có: Vào Settings của máy ảo > Network Adapter. Thay vì chọn Bridged, bạn có thể chọn Custom: Specific virtual network và chọn VMnet2 từ danh sách thả xuống.

Thêm card mạng mới: Một máy ảo có thể có nhiều card mạng. Bạn có thể giữ card mạng cũ kết nối với VMnet0 và thêm một card mạng thứ hai (Add > Network Adapter) cho máy ảo đó, sau đó cấu hình card mạng mới này để kết nối vào VMnet2.

##### Các kịch bản sử dụng thực tế

Việc tạo thêm các mạng ảo như VMnet2 rất hữu ích cho các công việc của một DevOps Engineer:

Tạo môi trường Lab đa mạng: Bạn có thể có một máy ảo đóng vai trò là router/firewall với hai card mạng: một card kết nối ra VMnet0 (mạng "public") và một card kết nối vào VMnet2 (mạng "private" hoặc "DMZ"). Các máy ảo khác sẽ kết nối vào VMnet2 và đi qua máy ảo router đó.

Tách biệt mạng quản lý: Một card mạng của máy ảo có thể nằm trong mạng Bridged để cung cấp dịch vụ, trong khi card mạng thứ hai nằm trong mạng Host-only (VMnet2) để bạn truy cập quản trị (SSH, RDP) một cách an toàn từ máy thật.

Kiểm thử không ảnh hưởng mạng chính: Tạo một mạng Host-only (VMnet2) hoàn toàn mới để dựng một cụm Kubernetes hoặc một môi trường test mà không làm ảnh hưởng đến dải IP của mạng công ty/nhà bạn.
