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
- Các thành phần chính của Pomerium:
  - Pomerium Proxy: là thành phần nhận request trực tiếp từ người dùng, đóng vai như một chốt chặn đầu tiên cho toàn bộ các services phía sau
  - Pomerium Authenticate: là thành phần giúp xác thực danh tính người dùng, quyết định xem người dùng có hợp lệ hay không
  - Pomerium Authorize: là thành phần giúp xác thực quyền của người dùng, đảm bảo chỉ những người dùng có đủ quyền mới được truy cập
  - Pomerium Broker: 



#### Quy trình xác thực khi truy cập qua Pomerium

<img width="848" height="606" alt="2025-10-05_171129" src="https://github.com/user-attachments/assets/99554850-1edd-4db6-8e34-dca20f013cec" />

- Client gửi request đến địa chỉ website cần truy cập thông qua Pomerium Proxy. Request này ban đầu chưa có session hoặc token xác thực.
- Proxy Pomerium kiểm tra cookie nếu client đã xác thực hay chưa. Nếu chưa sẽ redirect client đến dịch vụ xác thực Pomerium Authenticate Service.
- Pomerium Authenticate sẽ yêu cầu người dùng đăng nhập qua IdP (theo cấu hình, ví dụ Google/GitHub), thực hiện luồng xác thực OAuth hoặc OIDC chuẩn.
- Sau khi xác thực thành công, IdP trả về access token và thông tin người dùng cho dịch vụ Pomerium Authenticate. Pomerium Authenticate sinh ra session, lưu trữ dữ liệu trong Pomerium Databroker, và trả cho client một cookie bảo mật chứa session đã xác thực.
- Proxy Pomerium lấy thông tin session, truy vấn chính sách phân quyền (cấu hình policy), kiểm tra email, group, claim... xem người dùng có quyền với route này hay không. Nếu có, request mới được forward website

#### Cách setup Pomerium


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




