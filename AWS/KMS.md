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
