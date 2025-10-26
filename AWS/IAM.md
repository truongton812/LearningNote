# IAM

## Định nghĩa

- Root user là tài khoản gốc khi bạn tạo AWS account, chỉ có một duy nhất, dùng email và password đăng ký. Tất cả quyền quản trị cao nhất đều thuộc về root user này. Thông tin định danh (name) của root user chính là tên/email dùng khi tạo tài khoản, không có cấu trúc gì đặc biệt ngoài giá trị bạn nhập ban đầu.​
- IAM user là các user mà bạn tạo thêm để quản lý truy cập AWS. Mỗi IAM user có tên riêng (<name>), đi kèm theo account_id (ID tài khoản AWS) để phân biệt giữa các account. Đôi khi, bạn có thể cấu hình một "alias" để thay cho account_id, giúp đăng nhập dễ nhớ hơn thay vì dùng số ID dài.​ Định danh đầy đủ của IAM user thường có dạng <name>@<account_id>, hoặc <name>@<alias> nếu đặt alias cho account.
- Principal là đối tượng đại diện cho entity thực hiện hành động trên AWS, có thể là IAM user, federated user, IAM role, application. Group không được tính là principal, vì group chỉ dùng để gắn policy một cách tập trung nhưng không thể trực tiếp thực hiện hành động
- Các loại policy:
  - Managed policy: policy tạo sẵn bởi AWS
  - Custom policy: policy tự tạo
  - Inline policy: policy dành riêng cho 1 user (không hiển thị trong policy tab)

- Các loại based policy:
  - Identity-based policy
    - Là loại policy gắn trực tiếp cho identities như user, group, role. Policy này quy định user, group hoặc role đó được quyền làm gì trên các resource AWS.
    - Policy này không có trường "Principal", vì chính entity được gắn policy là principal rồi (vì đã biết policy gắn cho ai).​
    - Ví dụ: Gán policy cho user "A" để user này có quyền truy cập S3.
  - Resource-based policy
    - Là loại policy gắn trực tiếp cho resource (ví dụ: S3 bucket, SQS queue, KMS key...). Policy này quy định ai (principal nào) được quyền làm gì trên resource đó.
    - Policy này có trường "Principal" để chỉ rõ user, role, account nào được phép truy cập đến tài nguyên này.​
    - Ví dụ: Gán policy cho một S3 bucket để cho phép user khác account truy cập.
    - Lưu ý với resource-based policy cho S3, cần define policy ở cả bucket level và object level vì action cho object có thể cần quyền riêng biệt (ví dụ PUT, GET trên object).
  - Ví dụ minh họa:
    - Khi user muốn truy cập S3, có thể cấp quyền qua:
      - Identity-based policy: Gắn cho user policy "S3FullAccess" thì user sẽ truy cập được tất cả bucket/object cho phép trong policy đó.
      - Resource-based policy: Gắn trực tiếp policy vào S3 bucket để cấp/giới hạn quyền cho một hoặc nhiều principal từ các account khác hoặc role/application bất kỳ
      - lưu ý dù user có quyền truy cập thông qua identity-based policy, thì yêu cầu truy cập vẫn bị chặn nếu resource-based policy có một quy tắc Deny áp dụng cho user đó. (tham khảo thêm authorization process ở dưới)

- Cách add policy:
  - Inline policy: add trực tiếp vào user
  - Add qua group
  - Note: Nếu 1 user được gán quyền = inline và qua group thì sẽ có cả 2 quyền

## IAM role
- IAM role có 2 loại:
  - Service role: user define
  - Service link role: managed role do AWS tạo ra (không edit được)
- 1 IAM role có 2 thành phần:
  - Trust relationship: cho phép principal nào assume role (authen)
```
Effect: allow
Principal: { "service": "ec2.amazonaws.com" }
action: sts:assumeRole (luôn có)
```
  - Permission: cho phép làm gì sau khi assume (author)

Authorization Process:

<img width="935" height="312" alt="image" src="https://github.com/user-attachments/assets/ed545f4a-d8f6-49a4-8c6d-6d614e5a406a" />

AWS has always been adopting least permission policy across all the services. Meaning, if no permission about a specific resource is  specified, it would be a “Deny”. Post the initial deny decision, the final decision of deny or allow would be based on “Explicit” allows and denies specified in the permissions.

Ví dụ minh họa

1 user có 2 policy

Policy 1
```
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*"
}
```
Policy 2
```
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*",
  "Condition": {
    "IpAddress": {"aws:SourceIp": "102.102.102.102/32"},
    "NotIpAddress": {"aws:SourceIp": "111.111.111.111/32"}
  }
}
#Policy này chỉ allow nếu Source IP là 102.102.102.102 và đồng thời Source IP không phải là 111.111.111.111.
```

-> kết quả là nếu sử dụng source IP là 111.111.111.111 thì user sẽ ĐƯỢC phép dùng EC2 vì Policy 1 cho phép tất cả mà không có condition nào restrict IP, và Policy 2 chỉ là bổ sung thêm Allow với điều kiện chặt hơn. Không có explicit Deny xuất hiện, nên chỉ cần một Allow là đủ để pass

AWS Policy Evaluation: Nếu có nhiều policy cùng loại (chỉ toàn Allow, không có explicit Deny), chỉ cần một policy cho phép là user sẽ được phép thực hiện hành động, trừ khi có explicit Deny.


- Assume role: cho phép 1 principal assume quyền của 1 user / account khác, giúp người dùng không phải quản lý nhiều user cho các role khác nhau, giúp các service gọi đến các AWS service khác on your behalf.

- Permission boundary: giới hạn quyền của 1 user (overwrite attached permission). Note: không giúp Granted quyền

- Federation: cho phép dùng các user ở 1 hệ thống khác AWS (on-prem, azure,...) login vào AWS. Hỗ trợ 2 giao thức SAML 2.0 và web identity (google, facebook,...)

#### Hands-on
- Requirement: Acc A assume để có quyền của role ở acc B

Trên acc B:

Step 1: tạo policy (optional) hoặc dùng managed policy (VD truy cập S3, EC2,...)

Step 2: tạo role IAM: chọn trust entity là AWS account. Gán policy ở trên

Lưu ý ở bước chọn trust entity nếu chọn là AWS account thì mặc định sẽ grant account A quyền dùng dịch vụ STS:assumeRole của acc B . Còn nếu chọn custom trust policy thì phải define:

```
Effect: allow

Action: "STS:AssumeRole"

Principal: <account-A>
```
Trên acc A cần allow user được quyền dùng sts:assumeRole


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
