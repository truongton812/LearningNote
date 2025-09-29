# KMS

### Định nghĩa


- KMS có 2 cơ chế encrypt: symmetric và asymmetric

  - Symmetric: dùng 1 key để mã hóa và giải mã, không thể nhìn thấy symmetric key ở dạng unencrypted.

  - Asymmetric: dùng public key để mã hóa và private key để giải mã; public key có thể xem được, private key không thể xem ở dạng unencrypted.

Copy: Muốn share 1 AMI cho account khác thì snapshot của AMI ấy phải unencrypt = KMS key; không thể share nếu snapshot đấy được encrypt = default AWS-managed key.

Grant (Tác grant):

Grant: cấp/cung cấp quyền cho AWS principle truy cập KMS CMK, sử dụng token grant.

Grantee sẽ dùng token để encrypt/decrypt data.

Grant được gán bổ sung thông qua key policy. Vì key có thể có policy cho phép user A tạo grant:

text
{
  Effect: Allow
  Principal: { AWS: arn:aws:iam::123456:user/userA }
  Action: [ kms:CreateGrant, kms:ListGrant, kms:RevokeGrant ]
  Resource: *
}
User A có thể tạo grant cho user B, user B có thể dùng key X (nếu policy cho phép user B sử dụng).

Lệnh để tạo grant: aws kms create-grant --key-id <id>

grantee = principal <arn>, operations = "Encrypt" -> sẽ trả về GrantToken, user B dùng token này để encrypt data.
