Identity Aware Proxy (IAP)
Identity Aware Proxy (IAP) là một dịch vụ bảo mật chặn các yêu cầu đến ứng dụng web, xác thực người dùng tạo ra yêu cầu và chỉ cho phép các yêu cầu từ người dùng được ủy quyền có thể tiếp cận hệ thống. IAP thiết lập một lớp ủy quyền trung tâm cho các ứng dụng được truy cập qua HTTPS, cho phép sử dụng mô hình kiểm soát truy cập cấp ứng dụng thay vì chỉ dựa vào tường lửa cấp mạng. Thay vì yêu cầu VPN truyền thống, IAP kiểm soát quyền truy cập dựa trên danh tính người dùng và ngữ cảnh của yêu cầu như vị trí, thiết bị và vai trò.

IAP hoạt động bằng cách xác thực người dùng thông qua Dịch vụ nhận dạng của Google và thực hiện kiểm tra ủy quyền trước khi cho phép truy cập. Dịch vụ này có thể sửa đổi header của yêu cầu để bao gồm thông tin về người dùng đã xác thực, giúp ứng dụng nhận biết danh tính mà không cần tự mình thực hiện xác thực. IAP có thể bảo vệ các ứng dụng chạy trên nhiều nền tảng bao gồm App Engine, Compute Engine, Cloud Run và các dịch vụ đằng sau Google Cloud Load Balancer.

Các trường hợp sử dụng IAP
IAP phù hợp khi cần thiết lập quyền truy cập dựa trên nhóm, ví dụ một tài nguyên có thể truy cập được cho nhân viên nhưng không cho nhà thầu, hoặc chỉ truy cập được cho một phòng ban cụ thể. Tổ chức có thể chỉ định người dùng được xác thực bằng địa chỉ email công ty, địa chỉ Gmail, Google Workspace, hoặc địa chỉ từ các nhà cung cấp danh tính phổ biến khác. IAP còn có thể đặt ra chính sách cho thiết bị bao gồm yêu cầu về phiên bản hệ điều hành, sử dụng hồ sơ công ty, hoặc chỉ cho phép thiết bị thuộc sở hữu công ty.

Pomerium
Pomerium là một phần mềm mã nguồn mở hoạt động như Identity-Aware Proxy, được xây dựng dựa trên các nguyên tắc BeyondCorp và zero trust. Pomerium giúp bảo vệ các ứng dụng nội bộ, máy chủ và dịch vụ bằng cách xác minh liên tục danh tính người dùng, trạng thái thiết bị và ngữ cảnh yêu cầu trước khi cho phép truy cập. Không giống VPN truyền thống đã lỗi thời, Pomerium không yêu cầu phần mềm client riêng biệt và cung cấp giải pháp trung gian bảo vệ truy cập đa môi trường (đám mây, tại chỗ, đa đám mây).

Pomerium hỗ trợ xác thực qua nhiều nhà cung cấp identity provider như Google, Okta, GitHub và có thể kiểm soát truy cập đến cả các dịch vụ TCP như SSH, RDP, và cơ sở dữ liệu. Kiến trúc của Pomerium bao gồm các thành phần xử lý xác thực (Authenticate), phân quyền (Authorize), và lưu trữ dữ liệu phiên (Cache), giúp tổ chức triển khai mô hình zero trust một cách hiệu quả.




Quy trình xác thực khi truy cập qua Pomerium
Client gửi request đến verify.localhost.pomerium.io:
Trình duyệt truy cập domain nằm sau Pomerium. Request này ban đầu chưa có session hoặc token xác thực.

Pomerium kiểm tra session:
Proxy Pomerium sẽ kiểm tra cookie nếu client đã xác thực hay chưa. Nếu chưa, nó chuyển hướng (redirect) client đến dịch vụ xác thực Pomerium Authenticate Service.

Chuyển hướng đến IdP:
Dịch vụ Authenticate sẽ yêu cầu người dùng đăng nhập qua IdP (theo cấu hình, ví dụ Google/GitHub), thực hiện luồng xác thực OAuth hoặc OIDC chuẩn.

Nhận và lưu session:
Sau khi xác thực thành công, nhà cung cấp IdP trả về access token và thông tin người dùng cho dịch vụ Authenticate. Dịch vụ này sẽ sinh ra session, lưu trữ dữ liệu trong Pomerium Databroker, và trả cho client một cookie bảo mật chứa session đã xác thực.

Kiểm tra quyền truy cập:
Proxy Pomerium lấy thông tin session, truy vấn chính sách phân quyền (cấu hình policy), kiểm tra email, group, claim... xem người dùng có quyền với route này hay không. Nếu có, request mới được forward đến ứng dụng gốc (verify).

Forward request kèm thông tin xác thực:
Pomerium có thể thêm các identity header (như email, user id, thông tin OIDC claim, JWT...) vào request trước khi gửi về cho ứng dụng thật phía sau proxy. Ứng dụng verify nhận request này và biết rằng request đã được xác thực, không cần xử lý xác thực riêng nữa.

Minh họa flow
User → Pomerium Proxy → (nếu chưa xác thực: redirect) → Pomerium Authenticate → Identity Provider (Google/Github)

Khi thành công:
Pomerium Proxy kiểm tra session + policy → Ứng dụng verify
(request đi qua đã được xác thực bằng các header bổ sung)

---

Cách setup Pomerium


DNS: 
- trỏ CNAME authenticate.myapp.com → IP hoặc hostname của Pomerium Authenticate service.

- Main domain (myapp.com): trỏ A/ALIAS (hoặc CNAME) myapp.com → IP/hostname của Pomerium Proxy. Khi client truy cập https://myapp.com, request sẽ đến Pomerium Proxy -> Các request HTTPS tới myapp.com luôn đi qua Pomerium Proxy. Khi xác thực chưa có session, Pomerium Proxy sẽ redirect sang authenticate.myapp.com để login. Sau login, Pomerium Proxy forward request đến backend gốc (ví dụ service nội bộ trên port 8080).
- Lưu ý: Cả myapp.com và authenticate.myapp.com cần TLS.

Cấu hình Pomerium

pomerium.yaml

```
authenticate_service_url: https://authenticate.myapp.com
authorize_service_url:    https://authorize.myapp.com
shared_secret:            "<một chuỗi ngẫu nhiên để mã hóa cookie>"
idp_provider:             "google"
idp_client_id:            "<CLIENT_ID>"
idp_client_secret:        "<CLIENT_SECRET>"
# Định nghĩa route cho myapp.com
policies:
  - from: https://myapp.com
    to:   http://127.0.0.1:8080
    allowed_users:
      - "*@mycompany.com"
```


- Luồng truy cập

Client đi tới https://myapp.com.

Pomerium Proxy kiểm tra session cookie.

Nếu chưa tồn tại, redirect tới https://authenticate.myapp.com/oauth2/start?....

Client login tại Identity Provider.

Sau khi xác thực, client được redirect lại tới Pomerium Authenticate, sau đó trở về Proxy với session cookie.

Pomerium Proxy gọi dịch vụ Authorize (nội bộ) để kiểm tra policy.

Nếu được phép, Pomerium Proxy forward request đến ứng dụng gốc tại http://127.0.0.1:8080.
