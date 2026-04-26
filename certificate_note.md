
Lệnh để generate cert tự ký

openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -subj "/CN=192.168.1.100" -addext "subjectAltName = DNS:192.168.1.100,IP:192.168.1.100" -x509 -days 365 -out certs/domain.crt

Trong đó

openssl req: Sử dụng module req để tạo yêu cầu chứng chỉ/xin cấp chứng chỉ (certificate request) hoặc tạo chứng chỉ X.509.

-newkey rsa:4096: Tạo một cặp khóa mới dùng thuật toán RSA với độ dài 4096 bit.

-nodes: Không mã hóa khóa bí mật (key) bằng passphrase, phù hợp khi cần chạy dịch vụ tự động.

-sha256: Sử dụng thuật toán băm SHA-256 để ký chứng chỉ.

-keyout certs/domain.key: Đầu ra là file private key được lưu tại vị trí certs/domain.key.

-subj "/CN=192.168.1.100": Định nghĩa thông tin chủ thể (Distinguished Name), ở đây là Common Name (CN) là 192.168.1.100.

-addext "subjectAltName = DNS:192.168.1.100,IP:192.168.1.100": Thêm trường SAN (Subject Alternative Name) cho phép chứng chỉ dùng cho tên miền hoặc IP, ở đây là DNS và IP đều là 192.168.1.100.

-x509: Tạo chứng chỉ tự ký ngay, không phải chỉ yêu cầu cấp chứng chỉ (CSR).

-days 365: Chứng chỉ có hiệu lực trong 365 ngày.

-out certs/domain.crt: File chứng chỉ xuất ra ở certs/domain.crt.

Kết quả

Sau khi chạy xong lệnh này, bạn sẽ có hai file:

certs/domain.key: File private key (bí mật)

certs/domain.crt: Chứng chỉ số (certificate) đã bao gồm thông tin SAN, phù hợp cho dùng trong môi trường nội bộ, server dev/test hoặc container private registry, v.v.

Mục đích thực tế

Lệnh này hay dùng trong các trường hợp bạn cần kết nối bảo mật cho các dịch vụ trên LAN mà không cần mua chứng chỉ SSL từ CA bên ngoài, ví dụ: cài đặt registry Docker, ứng dụng test, nội bộ doanh nghiệp.

File domain.key là file khóa riêng tư (private key). Nó giữ vai trò rất quan trọng để mở khóa, giải mã dữ liệu được mã hóa gửi đến bạn và thực hiện các thao tác mã hóa/chữ ký số, bảo mật không cho người khác biết, đảm bảo rằng chỉ chủ sở hữu mới có thể sử dụng khóa này.

File domain.crt là file chứng chỉ số (certificate) chứa thông tin về chủ sở hữu (chẳng hạn tên miền hoặc IP), thông tin nhà cung cấp chứng chỉ và khóa công khai (public key). Nó được dùng để xác thực danh tính của server trên Internet và để các trình duyệt hoặc client tin tưởng kết nối bảo mật với server của bạn.

---

Mối liên hệ giữa domain.key và domain.crt

Khi trình duyệt kết nối HTTPS, nó nhận chứng chỉ domain.crt để kiểm tra tính hợp lệ, đồng thời dùng khóa công khai trong crt để mã hóa dữ liệu truyền.

Máy chủ dùng private key trong file domain.key để giải mã dữ liệu đã mã hóa từ client.

Cặp khóa này (private key + public key trong chứng chỉ) đảm bảo sự an toàn, bí mật dữ liệu và xác thực không bị giả mạo.

Các bước thực hiện:

Khi trình duyệt của bạn kết nối tới một trang web qua giao thức HTTPS, quá trình bảo mật dựa trên SSL/TLS diễn ra như sau liên quan đến file domain.crt (chứng chỉ số):

Trình duyệt yêu cầu kết nối bảo mật (SSL/TLS handshake) với server.

Server gửi file chứng chỉ domain.crt cho trình duyệt. File này chứa:

Khóa công khai (public key) của server.

Thông tin về server (tên miền, IP, tổ chức…).

Chữ ký số của một bên thứ ba đáng tin cậy (CA) hoặc tự ký trong trường hợp self-signed certificate. Chữ ký số trong chứng chỉ SSL/TLS là một đoạn dữ liệu mã hóa được tạo ra bởi cơ quan cấp chứng chỉ số (CA) hoặc bởi chính chủ sở hữu khi tự ký (self-signed). Nó giống như một dấu xác nhận danh tính số cho chứng chỉ đó và giúp trình duyệt hoặc các bên khác xác thực rằng chứng chỉ chưa bị sửa đổi, là thật và được cấp bởi nguồn đáng tin cậy. Khi trình duyệt nhận chứng chỉ, nó sẽ dùng khóa công khai của CA để kiểm tra chữ ký số này, nếu đúng thì chứng chỉ được xem là hợp lệ và tin cậy

Trình duyệt sẽ:

Kiểm tra tính hợp lệ của chứng chỉ: xác nhận chứng chỉ chưa hết hạn, tên miền có khớp, chứng chỉ được cấp bởi CA hợp lệ (hay với self-signed thì kiểm tra thủ công).

Nếu chứng chỉ hợp lệ, trình duyệt dùng khóa công khai trong domain.crt để mã hóa một khóa phiên (session key) dùng cho phiên giao tiếp này. Session key (khóa phiên) là một khóa bí mật tạm thời được tạo ra trong quá trình bắt tay SSL/TLS giữa trình duyệt và máy chủ. Khóa phiên này do phía trình duyệt (client) tạo ra và sau đó được mã hóa bằng khóa công khai trong chứng chỉ SSL (file domain.crt) của máy chủ để gửi cho máy chủ. Máy chủ sẽ dùng khóa riêng tư (private key, file domain.key) để giải mã lấy khóa phiên. Sau đó cả hai phía dùng khóa phiên này để mã hóa, giải mã dữ liệu trao đổi trong phiên đó, đảm bảo tính bảo mật và hiệu suất trong toàn bộ phiên làm việc

Server dùng private key trong domain.key để giải mã khóa phiên này. Khóa phiên được sử dụng để mã hóa và giải mã dữ liệu trao đổi trong suốt phiên làm việc, đảm bảo tính riêng tư và bảo mật.

Sau bước này, toàn bộ dữ liệu giữa trình duyệt và server được mã hóa, tránh bị nghe lén hoặc giả mạo.

---

Trao đổi dữ liệu bằng HTTPS

### Nền tảng
- Asymmetric (bất đối xứng) — dùng cặp khoá Public/Private:
  - Public Key  → mã hoá   (ai cũng có thể có)
  - Private Key → giải mã  (chỉ server giữ)
  - Encrypt(data, public_key)  → ciphertext
  - Decrypt(ciphertext, private_key) → data
- Symmetric (đối xứng) — dùng 1 khoá chung:
  - Encrypt(data, secret_key) → ciphertext
  - Decrypt(ciphertext, secret_key) → data
→ TLS dùng Asymmetric để trao đổi khoá, sau đó dùng Symmetric để mã hoá data thực.

- Certificate (.crt) là một file chứa:
  - Public Key của server
  - Domain name (Common Name): google.com
  - Thời hạn: 2024-01-01 → 2025-01-01
  - Chữ ký số của CA (Certificate Authority)
- CA (Certificate Authority) là bên thứ 3 được tin tưởng (Let's Encrypt, DigiCert...). CA ký vào cert để xác nhận "Public Key này đúng là của google.com". Browser có sẵn danh sách Root CA được tin tưởng. Khi nhận cert, browser verify chữ ký của CA → biết cert là thật.

TLS Handshake — luồng chi tiết
```
Browser                                    Server
  │                                          │
  │──── 1. ClientHello ───────────────────►  │
  │     (TLS version, cipher suites,         │
  │      client random)                      │
  │                                          │
  │  ◄─── 2. ServerHello ────────────────── │
  │     (chọn cipher suite,                  │
  │      server random)                      │
  │                                          │
  │  ◄─── 3. Certificate ────────────────── │
  │     (public key + domain + CA signature) │
  │                                          │
  │  ◄─── 4. ServerHelloDone ──────────────  │
  │                                          │
  │  5. Browser verify cert                  │
  │     - CA signature hợp lệ?              │
  │     - Domain khớp?                       │
  │     - Chưa hết hạn?                      │
  │                                          │
  │──── 6. ClientKeyExchange ─────────────► │
  │     Encrypt(pre-master secret,           │
  │             server_public_key)           │
  │                                          │
  │  7. Cả 2 bên tính Session Key:           │
  │     session_key = f(                     │
  │       client_random,                     │
  │       server_random,                     │
  │       pre-master_secret                  │
  │     )                                    │
  │                                          │
  │──── 8. ChangeCipherSpec ──────────────► │
  │──── 9. Finished (encrypted) ──────────► │
  │                                          │
  │  ◄─── 10. ChangeCipherSpec ──────────── │
  │  ◄─── 11. Finished (encrypted) ──────── │
  │                                          │
  │  ══════ Handshake xong ══════════════   │
  │                                          │
  │──── GET /index.html (encrypted) ──────► │
  │  ◄─── 200 OK + body (encrypted) ─────── │
```

- Bước quan trọng nhất: trao đổi khoá (bước 6-7). Pre-master secret là một số random do browser tạo ra. Browser encrypt nó bằng public key của server rồi gửi đi.
```
Browser tạo: pre_master = random()
Gửi: Encrypt(pre_master, server_public_key)

Server nhận và decrypt: Decrypt(ciphertext, server_private_key) → pre_master
```

- Kẻ nghe lén có thể thấy ciphertext nhưng không có private key → không decrypt được.

- Bây giờ cả 2 bên đều có:
  - client_random (gửi trong bước 1, public)
  - server_random (gửi trong bước 2, public)
  - pre_master_secret (chỉ 2 bên biết)

→ Tính ra Session Key giống nhau mà không cần gửi trực tiếp session key qua mạng.


Tóm tắt luồng
1. Handshake      → xác thực server, thoả thuận thuật toán
2. Key Exchange   → trao đổi secret bằng asymmetric crypto
3. Session Key    → derive symmetric key từ secret đó
4. Data Transfer  → mọi HTTP traffic encrypt bằng session key (AES/ChaCha20)

Asymmetric giải quyết bài toán trao đổi khoá an toàn.
Symmetric giải quyết bài toán mã hoá nhanh cho toàn bộ data.
Certificate + CA giải quyết bài toán xác thực danh tính server.

---

Dùng certbot để lấy chứng chỉ Let's Encrypt miễn phí tự động:

Certbot lấy chứng chỉ SSL miễn phí từ tổ chức uy tín Let's Encrypt, đảm bảo được trình duyệt công nhận là hợp lệ và không hiện cảnh báo bảo mật. Nó còn có khả năng tự động gia hạn chứng chỉ định kỳ, giúp bảo trì dễ dàng và liên tục.
```
# Cài đặt certbot (ví dụ trên Ubuntu)
sudo apt install certbot

# Mở cổng HTTP/HTTPS trên firewall
sudo ufw allow 80
sudo ufw allow 443

# Lấy chứng chỉ với standalone mode (ngắt dịch vụ web chạy trên cổng 80,443 khi lấy)
sudo certbot certonly --standalone -d yourdomain.com --preferred-challenges http --agree-tos --keep-until-expiring

# Certbot tạo các file chứng chỉ tại /etc/letsencrypt/live/yourdomain.com/
# và tự động cấu hình lịch gia hạn chứng chỉ
```

Certbot tạo ra chứng chỉ được trình duyệt công nhận, trong khi đó, openssl tạo ra chứng chỉ tự ký (self-signed certificate) hoàn toàn thủ công, không được CA nào xác nhận nên trình duyệt sẽ báo cảnh báo không tin cậy khi truy cập. Openssl chỉ phù hợp dùng cho thử nghiệm hoặc môi trường nội bộ.


---

Flow khi mua cert từ các bên thứ 3 (Let's Encrypt, DigiCert...):

1. Bạn tự generate cặp key: private_key + public_key

2. Bạn tạo CSR (Certificate Signing Request):
  - CSR = { domain, public_key, thông tin tổ chức }
  - CSR được ký bằng private_key của bạn (để CA verify bạn đúng là người giữ private_key)

3. Gửi CSR lên CA (Let's Encrypt, DigiCert...)

4. CA verify bạn sở hữu domain bằng 1 trong các cách:
- HTTP-01 Challenge: CA yêu cầu: đặt file tại `http://app.example.com/.well-known/acme-challenge/random_token`
- DNS-01 Challenge: CA yêu cầu: tạo TXT record `_acme-challenge.app.example.com = "random_value"`
- Email validation (DigiCert, Comodo...): CA gửi email đến admin@example.com hoặc whois contact. Bạn click confirm trong email → Chứng minh bạn là owner của domain

6. CA ký và trả về Certificate (.crt)

7. Bạn có: private_key (tự giữ) + cert (CA cấp)

