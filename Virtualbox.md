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
