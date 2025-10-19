# KMS

### Định nghĩa

- Là dịch vụ giúp encrypt và decrypt data
- KMS có 2 cơ chế encrypt: symmetric và asymmetric

  - Symmetric: dùng 1 key để mã hóa và giải mã, không thể nhìn thấy symmetric key ở dạng unencrypted.

  - Asymmetric: dùng public key để mã hóa và private key để giải mã; public key có thể xem được, private key không thể xem ở dạng unencrypted.

- Có 2 loại KMS key:

  - Key mà user tạo trong KMS gọi là CMK (customer master key). CMK contains key material dùng để encrypt/decrypt. By default, KMS tạo key material cho CMK, tuy nhiên ta cũng có thể "import key material" từ:

    - External

    - CloudHSM key store

    - External key store

    CMK key có thể encrypt data size max là 4 KB, nếu hơn phải dùng Data Encryption key.
    
    CMK có thể dùng để generate, encrypt và decrypt Data Encryption key.

  - AWS managed key là symmetric key do AWS quản lý, free, chỉ dùng cho dịch vụ tương ứng (VD: key aws/s3 chỉ dùng được cho S3). AWS managed key tự động được sinh ra trong account khi ta enable tính năng encryption của 1 service. Ta không thể manage các key này, không rotate hoặc đổi key policy được.

##### Tạo grant

- Grant cho phép 1 user cấp quyền cho AWS principle sử dụng KMS CMK bằng token -> Grantee sẽ dùng token để encrypt/decrypt data.

- Grant được gán trong key policy. Ví dụ key X có policy cho phép user A tạo grant:

```
{
  Effect: Allow
  Principal: { AWS: arn:aws:iam::123456:user/userA }
  Action: [ kms:CreateGrant, kms:ListGrant, kms:RevokeGrant ]
  Resource: *
}
```

=> User A có thể tạo grant cho user B, user B có thể dùng key X mà key X không cần policy cho phép user B sử dụng

- Lệnh để tạo grant: aws kms create-grant --key-id <id> --grantee-principal <arn> --operations = "Encrypt" -> sẽ trả về GrantToken, user B dùng token này để encrypt data.

### Các phương pháp để lưu trữ database password trong AWS

Bạn có thể sử dụng một trong các dịch vụ sau của AWS để lưu trữ database passwords một cách an toàn:

1. AWS Secrets Manager
- Dịch vụ quản lý secrets chuyên dụng của AWS, giúp lưu trữ database passwords, API keys, credentials, v.v.
- Tự động xoay vòng mật khẩu cho RDS, Redshift, DocumentDB, v.v.
- Mã hóa bằng AWS KMS (Key Management Service).
- Dễ dàng tích hợp với Lambda, EC2, ECS, EKS, v.v.
- Truy xuất mật khẩu bằng API hoặc AWS SDK mà không cần lưu trực tiếp vào mã nguồn.

AWS Secrets Manager không giới hạn chỉ dùng cho database của AWS như RDS hay Redshift. Bạn hoàn toàn có thể sử dụng Secrets Manager để lưu trữ và quản lý mật khẩu cho database cài trên EC2 hoặc bất kỳ hệ thống nào khác.

2. AWS Systems Manager Parameter Store
- Lưu trữ database passwords dưới dạng SecureString (mã hóa bằng KMS).
- Miễn phí đến 10.000 parameters (Secrets Manager có phí cao hơn).
- Truy xuất qua API, SDK hoặc CLI.
- không tự động xoay vòng mật khẩu như Secrets Manager.

3. AWS KMS (Key Management Service) + S3 hoặc DynamoDB
- Lưu mật khẩu database trong DynamoDB hoặc S3, nhưng mã hóa bằng AWS KMS.
- Không phải giải pháp chuyên dụng như Secrets Manager nhưng có thể dùng trong một số trường hợp. Ví dụ: Mã hóa password trước khi lưu vào DynamoDB

### so sánh AWS Secrets Manager và AWS Systems Manager Parameter Store (thường gọi tắt là Parameter Store)

1. Mục đích sử dụng
- Parameter Store: là một kho lưu trữ phân cấp an toàn, dùng để quản lý cấu hình ứng dụng, biến môi trường, khóa sản phẩm, URL, host database, và cả secrets (dữ liệu nhạy cảm có thể mã hóa). Nó được thiết kế cho cả dữ liệu cấu hình thường và bí mật.

- Secrets Manager: tập trung chuyên sâu vào quản lý secrets như mật khẩu, API keys, token, với các tính năng nâng cao dành cho bảo mật như tự động xoay vòng (rotate) secrets, tạo mật khẩu ngẫu nhiên.

2. Giới hạn kích thước dữ liệu
- Parameter Store: chuẩn thường cho phép 4KB cho mỗi giá trị, loại nâng cao (advanced) cho phép tối đa 8KB.

- Secrets Manager: cho phép lưu trữ secrets lên đến 64KB.

3. Tính năng bảo mật và quản lý
- Cả hai đều hỗ trợ mã hóa bằng AWS KMS.

- Secrets Manager có chức năng tự động xoay vòng secrets để tăng cường bảo mật.

- Parameter Store mã hóa có thể bật hoặc tắt tùy chọn, còn Secrets Manager mặc định có bảo mật nâng cao.

- Parameter Store hỗ trợ version cho từng parameter, nhưng mỗi thời điểm chỉ có 1 phiên bản active, còn Secrets Manager hỗ trợ nhiều version của secret cùng lúc.

4. Chi phí
- Parameter Store có phiên bản chuẩn miễn phí với giới hạn 10,000 parameters, phiên bản nâng cao có phí phụ thu theo lượt gọi API.

- Secrets Manager tính phí dựa trên số lượng secrets và lượt truy cập API, thường cao hơn do tính năng tự động xoay vòng.

5. Khả năng mở rộng và đồng bộ
- Secrets Manager hỗ trợ sao chép secrets giữa các vùng (cross-region replication) một cách dễ dàng.

- Parameter Store không hỗ trợ replica cross-region out-of-the-box.

---


# Key Management Service (KMS)

## I. Định nghĩa và khái niệm
- Là dịch vụ của Amazon Web Services giúp tạo, lưu trữ, và quản lý các khóa mã hóa một cách an toàn với độ bền và bảo mật cực cao. S
- Các khóa mã hóa của KMS có thể dùng để mã hóa (encrypt), giải mã (decrypt) hoặc ký số (sign) dữ liệu được lưu trữ hoặc truyền qua các dịch vụ khác của AWS như S3, EBS, RDS hay Secrets Manager.​



- KMS key sử dụng hai cơ chế mã hóa:
  - Symmetric Keys (Khóa đối xứng)
  - Sử dụng một khóa duy nhất để thực hiện cả mã hóa và giải mã dữ liệu.
  - Loại khóa này dùng thuật toán AES-256, rất bảo mật và phổ biến.
  - Các dịch vụ AWS tích hợp với KMS đều sử dụng loại khóa đối xứng này để mã hóa dữ liệu (ví dụ: S3, EBS, RDS).
  - AWS không cho phép người dùng truy cập trực tiếp vào key này, bạn chỉ dùng API KMS để mã hóa/giải mã.
  - Symmetric key rất cần thiết cho mô hình Envelope Encryption: trong đó KMS dùng để mã hóa "key dữ liệu" trong khi dữ liệu lớn được mã hóa ngoài KMS.
  - Asymmetric Keys (Khóa bất đối xứng)
  - Bao gồm một cặp khóa: Public Key (khóa công khai) dùng để mã hóa hoặc xác thực, và Private Key (khóa riêng tư) dùng để giải mã hoặc ký số.
  - Các thuật toán hỗ trợ: RSA (2048/3072/4096-bit) và ECC (Elliptic Curve Cryptography).
  - Private Key được sinh và bảo vệ trong AWS KMS, không bao giờ bị lộ ra ngoài. AWS quản lý việc sử dụng private key qua API, bạn không thể tự lấy private key ra.
  - Public Key có thể tải xuống và sử dụng ngoài AWS (ví dụ cho các ứng dụng bên ngoài không gọi được API KMS).
  - Dùng cho các mục đích như encrypt/decrypt bên ngoài AWS hoặc thực hiện các thao tác ký và xác thực.
  - Nếu bạn cần mã hóa hoặc xác thực dữ liệu ngoài AWS, hoặc cho ứng dụng không gọi được KMS API, bạn dùng asymmetric key vì có thể cho người khác biết public key để mã hóa hoặc kiểm tra chữ ký.



- Khi tạo Customer Managed Key (CMK) trong AWS KMS có thể chỉ định thuộc tính key material origin, dùng để xác định nguồn gốc "key material" cho KMS key đó.​
- Key material là dãy số hoặc chuỗi bí mật (ví dụ 256 ký tự nếu là AES-256) mà tất cả các quá trình mã hóa, giải mã sẽ dùng. Key hiển thị trên AWS chỉ dùng để quản lý ( có các thuộc tính như tên key, ID, policy quản lý), nhưng phải có “key material” (bản thân dữ liệu khóa đó) thì AWS mới thực hiện được mã hóa/giải mã.
- AWS sẽ tự sinh key material giúp bạn khi tạo khóa, hoặc bạn tự upload/nhập vào nếu muốn kiểm soát tuyệt đối nguồn gốc của khóa đó.
- Một số tùy chọn phổ biến gồm:
  - AWS_KMS: AWS sẽ tự động sinh và quản lý key material (mặc định, khuyến nghị). Bạn không thấy hay thao tác với dữ liệu khóa gốc, tất cả được bảo vệ bởi phần cứng HSM đạt chuẩn FIPS.
  - EXTERNAL (Import key material): Bạn chủ động sinh khóa bí mật ngoài AWS (bằng thiết bị HSM hoặc phần mềm tuân thủ quy định riêng) và upload vào AWS KMS. Dùng khi có nhu cầu tự kiểm soát vòng đời hoặc tuân thủ quy định đặc biệt về kiểm soát/nguồn gốc khóa.
  - AWS_CLOUDHSM: Key material được sinh trong CloudHSM cluster do bạn quản lý trên AWS, phù hợp khi cần toàn quyền kiểm soát lifecycle và access audit.
  - EXTERNAL_KEY_STORE: Key material nằm hoàn toàn ngoài AWS (ở một external key manager/HSM do bạn sở hữu), dùng với dịch vụ Custom Key Store, đáp ứng các yêu cầu kiểm soát vật lý hoặc kỹ thuật nghiêm ngặt nhất.


---



Các loại key trong AWS KMS:
- AWS Managed Keys (khóa do AWS quản lý)
  - AWS tự động tạo và quản lý loại khóa này cho bạn khi bật mã hóa mặc định trên các dịch vụ (S3, EBS, RDS, v.v.).​
  - Bạn không thể truy cập trực tiếp khóa nhưng không cần lo quản lý vòng đời hay chính sách quyền truy cập.
  - Được đặt tên theo định danh dạng aws/service-name (ví dụ: aws/s3).
- Customer Managed Keys (CMK - khóa do người dùng tạo và quản lý)
  - Người dùng có thể tạo, cấu hình, gắn quyền truy cập, kích hoạt rotation, hoặc xóa khóa này.​
  - Có thể thiết lập automatic key rotation hằng năm.
  - Phù hợp cho các ứng dụng yêu cầu kiểm soát chi tiết hoặc tuân thủ quy định bảo mật.

