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

=> User A có thể tạo grant cho user B, user B có thể dùng key X mà key X không cần policy cho phép user B sử dụng

- Lệnh để tạo grant: aws kms create-grant --key-id <id> --grantee-principal <arn> --operations = "Encrypt" -> sẽ trả về GrantToken, user B dùng token này để encrypt data.
