# Các chế độ của card mạng trong Virtualbox

## Các options cho card mạng trong VirtualBox
1. NAT (Network Address Translation):
Máy ảo có thể truy cập Internet thông qua card mạng của máy chủ vật lý, nhưng các máy khác trong mạng không truy cập được vào máy ảo. Đây là chế độ mặc định.

2.	Bridged Adapter:
Máy ảo sẽ kết nối trực tiếp vào mạng vật lý như một máy tính thật, nhận IP từ DHCP của mạng LAN. Máy ảo và máy thật có thể nhìn thấy nhau và các thiết bị khác trong mạng.

3. NAT Network:
Cho phép nhiều máy ảo giao tiếp với nhau qua một mạng NAT riêng biệt, đồng thời vẫn truy cập được Internet.

4. Internal Network:
Tạo một mạng ảo nội bộ, chỉ các máy ảo cùng cấu hình Internal Network mới giao tiếp được với nhau. Máy chủ vật lý không truy cập được vào mạng này.

5. Host-Only Adapter:
Tạo một mạng riêng giữa máy chủ vật lý và các máy ảo, không truy cập được ra ngoài Internet. Thường dùng để test hoặc phát triển

6. Generic Driver:
Chế độ đặc biệt, ít dùng, cho phép sử dụng driver mạng tùy chỉnh hoặc các extension đặc biệt.

8. Not Attached:
Card mạng được tạo ra nhưng không kết nối với bất kỳ mạng nào, giống như rút dây mạng ra khỏi card.

## Mối liên hệ giữa card mạng trên máy ảo và máy thật


Khi tạo card mạng (network adapter) cho máy ảo trong VirtualBox, không phải lúc nào trên máy host cũng sinh ra một card mạng tương ứng. Việc này phụ thuộc vào loại chế độ mạng (network mode) mà bạn chọn cho card mạng của máy ảo:
| Chế độ card mạng trên máy ảo | Card mạng mới trên host có sinh ra không? | Ghi chú |
| :--- | :--- | :--- |
| NAT | Không | Máy ảo dùng chung kết nối mạng của host, không tạo card riêng trên host |
| Bridged Adapter | Không | Máy ảo kết nối trực tiếp vào mạng vật lý qua card thật của host. |
| Host-Only Adapter | Có | Host sẽ sinh ra một card mạng ảo, thường tên là “VirtualBox Host-Only Network” |
| Internal Network | Không | Chỉ tạo mạng ảo giữa các máy ảo, host không tham gia. |
| NAT Network | Có (nếu chưa có) | Khi tạo NAT Network mới, VirtualBox sẽ tạo một card mạng ảo dùng chung cho các máy ảo trong NAT Network đó. |
| Not Attached | Không | Không kết nối vào đâu cả.


## Cấp phát IP cho máy ảo
#### Ở chế độ Bridged Adapter
- Khi cấu hình card mạng của máy ảo VirtualBox ở chế độ Bridged Adapter, IP của máy ảo sẽ do DHCP server của mạng thật (thường là router hoặc switch trong mạng LAN) cấp phát nếu bạn để máy ảo ở chế độ nhận IP động (DHCP)

#### Ở chế độ NAT
- Khi bạn cấu hình card mạng của máy ảo VirtualBox ở chế độ NAT, máy ảo sẽ nhận được IP riêng trong một mạng ảo do VirtualBox tạo ra. Cụ thể:
    - Địa chỉ IP mặc định của máy ảo là 10.0.2.15.
    - Gateway mặc định là 10.0.2.2.
    - DHCP server mặc định là 10.0.2.3.

- Mỗi máy ảo ở chế độ NAT sẽ có một mạng ảo riêng biệt, nên nếu bạn chạy nhiều máy ảo cùng lúc ở chế độ này, mỗi máy vẫn sẽ nhận được IP 10.0.2.15 nhưng trong các mạng ảo tách biệt - chúng không nhìn thấy nhau qua NAT.

- Nếu bạn muốn các máy ảo giao tiếp với nhau qua NAT (tức là cùng một subnet), bạn cần dùng chế độ NAT Network thay vì NAT thông thường. Khi đó, các máy ảo sẽ được cấp phát IP trong dải bạn cấu hình cho NAT Network (ví dụ: 10.0.2.x, 10.1.1.x, v.v.) và có thể giao tiếp với nhau

- Lưu ý :  Khi bạn sử dụng chế độ NAT (hoặc NAT Network) cho card mạng của máy ảo trong VirtualBox, bạn nên để máy ảo nhận IP động (DHCP). VirtualBox sẽ tự động cấp phát IP cho máy ảo thông qua DHCP server tích hợp sẵn trong phần mềm. Dải IP default mà VirtualBox cấp cho các máy ảo có thể thay đổi nếu dùng chế độ NAT Network và Host-Only Adapter. Chế độ NAT mặc định không cho phép thay đổi dải IP qua giao diện đồ họa, nhưng có thể tinh chỉnh nâng cao bằng dòng lệnh VBoxManage. Tuy nhiên, cách này phức tạp và ít được sử dụng hơn so với NAT Network. Cách làm:
    - Mở VirtualBox, vào File > Preferences > Network > NAT Networks.
    - Chọn NAT Network bạn muốn chỉnh sửa -> Edit.
    - Tại mục Network CIDR, nhập dải IP mong muốn, ví dụ: 192.168.100.0/24 thay vì 10.0.2.0/24.


## SSH vào máy ảo khi ở chế độ NAT

#### Nguyên nhân bạn không SSH được vào máy ảo VirtualBox với chế độ NAT
- Khi bạn chọn chế độ NAT cho card mạng của máy ảo, VirtualBox sẽ tạo một thiết bị NAT ảo, hoạt động giống như một router riêng biệt giữa máy ảo và máy host. 
Địa chỉ của thiết bị NAT ảo này là 10.0.2.2, cũng là địa chỉ gateway mặc định mà máy ảo sử dụng
Vai trò của thiết bị NAT ảo là:
    - Cung cấp một mạng riêng ảo cho máy ảo, cấp phát địa chỉ IP (thường là 10.0.2.15) thông qua DHCP và
    - Định tuyến lưu lượng giữa máy ảo và mạng ngoài: khi máy ảo gửi dữ liệu ra ngoài Internet, thiết bị NAT ảo này sẽ nhận dữ liệu từ máy ảo, sau đó chuyển tiếp dữ liệu đó ra ngoài thông qua card mạng vật lý của máy host. 
- Thiết bị NAT ảo này sẽ thực hiện chức năng dịch địa chỉ (NAT), cho phép máy ảo truy cập Internet nhưng kết nối trực tiếp từ bên ngoài không thể đến được máy ảo. Do NAT chỉ cho phép kết nối một chiều: Máy ảo gửi gói tin ra ngoài, NAT sẽ ghi nhớ và cho phép trả lời quay lại. Nhưng nếu có một kết nối mới từ ngoài vào (ví dụ: bạn SSH từ host vào máy ảo), NAT sẽ không biết chuyển gói tin này đến đâu, nên sẽ chặn lại.

#### Cách để SSH vào máy ảo khi dùng chế độ NAT
- Để thực hiện SSH vào máy ảo ở chế độ NAT, cần cấu hình chuyển tiếp cổng (port forwarding). Port forwarding là cách bạn "mở một cánh cửa" trên máy host, chuyển tiếp các kết nối đến một cổng cụ thể (ví dụ: 2222) vào đúng cổng dịch vụ (ví dụ: 22/SSH) trên máy ảo
- Cách làm: 
    - Tắt máy ảo
    - Mở VirtualBox, chọn máy ảo cần cấu hình. Vào Settings → Network → Adapter 1 (đảm bảo đang ở chế độ NAT). Nhấn Advanced → Port Forwarding.
    - Thêm một rule mới với thông tin như sau:
        - Name: SSH
        - Protocol: TCP
        - Host IP: 127.0.0.1 (hoặc để trống)
        - Host Port: 2222 (hoặc 2522, miễn là chưa bị chiếm dụng)
        - Guest IP: IP của máy ảo (thường là 10.0.2.15, kiểm tra bằng lệnh ip a trong máy ảo)
        - Guest Port: 22 (mặc định của SSH)
    - Sau khi đã cấu hình port forwarding, bạn có thể SSH vào máy ảo bằng lệnh:
    ```bash
        ssh -p 2222 <user>@127.0.0.1
    ```
- Khi thêm rule để thực hiện port-forwarding trong VirtualBox với chế độ NAT, bạn đang thêm rule cho thiết bị NAT ảo. Rule port-forwarding giúp thiết bị NAT ảo biết rằng: “Nếu có kết nối đến cổng X trên máy host, hãy chuyển tiếp (forward) kết nối đó đến cổng Y trên máy ảo.”
