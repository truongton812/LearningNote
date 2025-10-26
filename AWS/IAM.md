# IAM
## IAM - Passrole
- là permission cho phép 1 principle (IAM user, IAM role, AWS service) chuyển giao (pass) 1 IAM role cho 1 dịch vụ AWS khác
- VD:
  - User A có quyền tạo lambda function hoặc EC2.
  - Lambda function hoặc EC2 cần sử dụng 1 role X nào đó để thao tác -> User A cần phải có quyền passrole X
- Note: passrole không cho phép principle dùng role đấy, nó chỉ cấp quyền chuyển giao
- Example
```
Effect: allow,
action: iam:passrole,
Resource: arn:aws:iam:123456:role/Myrole
```
-> sau đó đem cái này gán cho principle
