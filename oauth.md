OAuth 2.0 là một giao thức ủy quyền (authorization) chứ không phải giao thức xác thực (authentication) thuần túy, dùng để cho phép ứng dụng bên thứ ba (client) được truy cập tài nguyên của người dùng trên một hệ thống khác (ví dụ: website, API) mà không cần biết mật khẩu của họ. Dưới đây là mô tả luồng phổ biến nhất: Authorization Code Flow – tức luồng từ “client” (web app) đến “website” (authorization server + resource server).

Các thành phần chính trong OAuth
Resource Owner (người dùng): người sở hữu tài nguyên (ví dụ: dữ liệu trên Google, Facebook, v.v.).

Client (application): website/app của bạn (ví dụ: một web app cần đăng nhập bằng Google).

Authorization Server: nơi cho phép/ủy quyền, xác thực username/password và trả về access token. Ví dụ: Google, Facebook, GitHub.

Resource Server: nơi chứa tài nguyên (API, file, user info…) và kiểm tra access token trước khi trả dữ liệu.

Luồng xác thực OAuth từ client đến website (Authorization Code Flow)
Client chuyển hướng người dùng đến Authorization Server

Người dùng truy cập ứng dụng web (client) và bấm “Đăng nhập bằng Google/Facebook/…”.

Client chuyển hướng người dùng đến URL login của Authorization Server, kèm:

client_id,

redirect_uri (đường callback),

scope (quyền gì cần xin),

state (dùng chống CSRF).

Người dùng xác thực và cấp quyền

Trình duyệt mở trang login của Authorization Server.

Người dùng nhập tài khoản/mật khẩu và đồng ý cho client truy cập (Accept/Cancel).

Authorization Server xác thực người dùng và kiểm tra scope.

Authorization Server trả về authorization_code cho client

Nếu đồng ý, Authorization Server redirect trình duyệt về redirect_uri của client với:

code (authorization code – mã tạm thời),

state (nếu có).

Code này chỉ dùng được một lần và có thời gian sống ngắn.

Client dùng authorization_code để xin access_token

Client (ứng dụng web) gửi request đến điểm cuối token của Authorization Server, kèm:

client_id,

client_secret,

redirect_uri,

code vừa nhận được.

Authorization Server xác thực:

client_id/client_secret,

redirect_uri,

code còn hiệu lực.

Authorization Server trả về access_token (và có thể refresh_token)

Nếu hợp lệ, Authorization Server trả về:

access_token (dùng để gọi API),

refresh_token (tùy luồng, để gia hạn token sau này).

Client lưu access_token (thường trong session, session storage, hoặc gửi tiếp lên backend).

Client dùng access_token truy cập tài nguyên trên Resource Server

Client gửi request tới Resource Server (API) với header như:

Authorization: Bearer <access_token>.

Resource Server kiểm tra access_token với Authorization Server hoặc tự validate (JWT), rồi trả về dữ liệu nếu token hợp lệ.

Tóm tắt ngắn cho DevOps / backend
Client không bao giờ thấy mật khẩu người dùng, chỉ làm việc với authorization_code và access_token.

Luồng này thường dùng cho web app (server‑side), vì client secret được giữ trên server, an toàn hơn so với client-side (SPA dùng PKCE).

Website (client) chủ yếu:

gửi redirect đến Authorization Server,

nhận code qua callback,

đổi code lấy token ở backend,

dùng token để gọi API.

Nếu bạn muốn, có thể mình mô tả thêm luồng dành cho SPA (frontend) hoặc microservices (client credentials, service account) để so sánh với luồng nói trên.


https://viblo.asia/p/tim-hieu-ve-co-che-xac-thuc-oauth2-aWj53W9856m


---

- User đăng nhập vào website, website kiểm tra có valid không. Nếu valid thì sẽ sinh ra session token và lưu vào db, đồng thời trả session token về cho client
- Client lưu token đấy vào cookie. Các request về sau sẽ gửi kèm session token
- Server kiểm tra xem session token có hợp lệ không bằng cách so sánh với db và phản hồi cho client nếu token hợp lệ

---

Các loại grant type: authorization code, implicit, client credentials và refresh token. Chọn grant type bằng cách vào App client -> Login pages -> Managed login pages configuration -> OAuth 2.0 grant types

Để biết login page có phải là authorization code grant_type không thì check ở URL nếu respond_type=code thì là authorzation code, nếu respond_type=token thì là implicit (lưu ý nếu ta chọn 2 grant type thì login page luôn sử dụng authorization code grant_type, ta cần phải manual sửa respond_type thành token nếu muốn sử dụng implicit grant_type)

Sau khi login thành công thì cognito trả về callback URL + code (duration có giới hạn, set ở `Appclient` -> `Authentication flow session duration`)

Sau khi có code ta có thể generate ra được token bằng cách gọi đến cognito endpoint (lấy ở Branding -> Domain -> Cognito domain có format dạng https://ap-northeast-17ktlpz92c.auth.ap-northeast-1.amazoncognito.com)

curl -X POST \
  https://<cognito_domain>/oauth2/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=authorization_code&client_id=<CLIENT_ID_HERE>&code=<AUTHORIZATION_CODE_HERE>&redirect_uri=<REDIRECT_URI_HERE>'

Các thông số CLIENT_ID_HERE lấy ở Appclient -> Client ID, AUTHORIZATION_CODE_HERE là code trả về sau khi login thành công, REDIRECT_URI_HERE là callback URL

-> trả về id_token (chứa thông tin user), access_token (chứa các thông tin dành cho API như group, client_id, iss...) và refresh_token (refresh_token là token dùng để get lại id_token và access_token nếu 2 token đấy hết hạn) . Lưu ý 1 code chỉ gọi đc 1 lần, gọi lần thứ 2 với cùng code sẽ báo lỗi invalid_grant

Lưu ý là id_token và access_token cũng theo format của JWT token

JWT token có 3 phần chính: header, payload và signature

Nếu dùng respond_type=implicit thì id_token và access_token sẽ trả luôn trong callback URL (không nên dùng loại này)

---

Luồng hoạt động khi app tích hợp với Cognito

<img width="1610" height="891" alt="image" src="https://github.com/user-attachments/assets/319eb391-e372-4568-8ba6-8566c5d4524f" />

- User login sẽ được redirect đến Cognito login UI, nếu login thành công thì Cognito sẽ sử dụng private để generate ra JWT token và trả lại cho client
- Client lưu JWT token vào cookie. Sau đó mỗi request tới server sẽ đính kèm JWT token
- Bản thân server không có key nên không thể xác thực được JWT token, server sẽ gọi tới Cognito endpoint để lấy public key (chỉ gọi 1 lần và lưu vào đâu đó hay là lần nào validate JWT token cũng gọi, cần check lại), nếu xác thực JWT token hợp lệ thì trả lại respond cho client

Để xem public key của Cognito thì vào Overview -> Token signing key URL 
