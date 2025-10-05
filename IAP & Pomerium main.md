## Identity Aware Proxy (IAP)

#### Định nghĩa

- Identity Aware Proxy (IAP) là dịch bảo mật dùng để kiểm tra các request đến website và chỉ cho phép các request đã được xác thực truy cập vào website. 
- IAP hoạt động như một reverse proxy, tuy nhiên thay vì tập trung vào việc cân bằng tải đến backend thì IAP tập trung vào việc xác thực người dùng
- IAP có thể kiểm soát quyền truy cập dựa trên danh tính người dùng hoặc các thông tin khác như vị trí, loại thiết bị,...

#### Các trường hợp sử dụng IAP
- IAP phù hợp khi cần thiết lập quyền truy cập dựa trên danh tính, ví dụ có thể sử dụng IAP để đảm bảo 1 website chỉ có thể truy cập được bằng tài khoản LDAP của công ty, hoặc từ những domain được định nghĩa trước, hoặc địa chỉ từ các IdP phổ biến khác như Gmail, Facebook, Github,...
- Ngoài danh tính còn có thể thiết lập các chính sách như chỉ cho phép thiết bị di động IOS, Android, hoặc địa chỉ IP từ 1 số vùng nhất định truy cập,...

## Pomerium

- Pomerium là một phần mềm opensource giúp triển khai Identity-Aware Proxy

<img width="848" height="606" alt="2025-10-05_171129" src="https://github.com/user-attachments/assets/99554850-1edd-4db6-8e34-dca20f013cec" />


In practice:

Authenticate: Users sign in through your identity provider.
Authorize: Pomerium checks policies to decide who gets access.
Proxy: Traffic to internal apps flows through a secure, policy-enforced route.
This approach simplifies managing access to internal services—no more network-level trust. Instead, trust is tied to identity, context, and a dynamic access policy.




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

---

