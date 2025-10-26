# IAM

## Định nghĩa

- Root user là tài khoản gốc khi bạn tạo AWS account, chỉ có một duy nhất, dùng email và password đăng ký. Tất cả quyền quản trị cao nhất đều thuộc về root user này. Thông tin định danh (name) của root user chính là tên/email dùng khi tạo tài khoản, không có cấu trúc gì đặc biệt ngoài giá trị bạn nhập ban đầu.​
- IAM user là các user mà bạn tạo thêm để quản lý truy cập AWS. Mỗi IAM user có tên riêng (<name>), đi kèm theo account_id (ID tài khoản AWS) để phân biệt giữa các account. Đôi khi, bạn có thể cấu hình một "alias" để thay cho account_id, giúp đăng nhập dễ nhớ hơn thay vì dùng số ID dài.​ Định danh đầy đủ của IAM user thường có dạng <name>@<account_id>, hoặc <name>@<alias> nếu đặt alias cho account.
- Principal là đối tượng đại diện cho entity thực hiện hành động trên AWS, có thể là IAM user, federated user, IAM role, application. Group không được tính là principal, vì group chỉ dùng để gắn policy một cách tập trung nhưng không thể trực tiếp thực hiện hành động

#### - Các loại policy:

Identity-based policies – jLà loại policy gắn trực tiếp cho identities như user, group, role. Policy này quy định user, group hoặc role đó được quyền làm gì trên các resource AWS. Policy này không có trường "Principal", vì chính entity được gắn policy là principal rồi (vì đã biết policy gắn cho ai).​ Có định dạng JSON và được quản lý theo 3 nhóm:
  - Managed policies là các chính sách độc lập bạn có thể sử dụng cho nhiều Identity. Được chia thành 2 loại
    - AWS managed policies: các chính sách do aws tạo và quản lý.
    - Customer managed policies: các chính sách do người dùng tạo và quản lý
  - Inline policies các chính sách do người dùng tạo và chị được gắn với một Identity duy nhất và sẽ bị xoá khi identity bị xoá.
- Ví dụ: Gán policy cho user "A" để user này có quyền truy cập S3. 


Resource-based policy: json policy permission là một inline policy được gắn với 1 resource, một resource-based policies quy định các principal được áp dụng chính sách và sử dụng condition để kiểm soát truy cập. Nó cho phép các principal ngoài account sử dụng các tài nguyên trong account. Đặc điểm:

Khi resource-based policy chỉ định 1 principal là một principal của một account khác, principal vẫn cần được cấp phép sử dụng các resource bởi Identity-based policy của account sở hữa principal (tham khảo ví dụ)

IAM có 1 loại resource-based policy là trust role policy được gắn cho IAM role cho phép role chỉ định Trusted entities và thực hiện cross-account access.

IAM roles và resource-based policy vẫn có sự khác biệt trong cơ chế ủy quyền cross-account access

- Là loại policy gắn trực tiếp cho resource (ví dụ: S3 bucket, SQS queue, KMS key...). Policy này quy định ai (principal nào) được quyền làm gì trên resource đó.
- Policy này có trường "Principal" để chỉ rõ user, role, account nào được phép truy cập đến tài nguyên này.​
- Ví dụ: Gán policy cho một S3 bucket để cho phép user khác account truy cập.
- Lưu ý với resource-based policy cho S3, cần define policy ở cả bucket level và object level vì action cho object có thể cần quyền riêng biệt (ví dụ PUT, GET trên object).

  
  
  Ví dụ minh họa:
    - Khi user muốn truy cập S3, có thể cấp quyền qua:
      - Identity-based policy: Gắn cho user policy "S3FullAccess" thì user sẽ truy cập được tất cả bucket/object cho phép trong policy đó.
      - Resource-based policy: Gắn trực tiếp policy vào S3 bucket để cấp/giới hạn quyền cho một hoặc nhiều principal từ các account khác hoặc role/application bất kỳ
      - lưu ý dù user có quyền truy cập thông qua identity-based policy, thì yêu cầu truy cập vẫn bị chặn nếu resource-based policy có một quy tắc Deny áp dụng cho user đó. (tham khảo thêm authorization process ở dưới)

- Cách add policy:
  - Inline policy: add trực tiếp vào user
  - Add qua group
  - Note: Nếu 1 user được gán quyền = inline và qua group thì sẽ có cả 2 quyền
 


Ngoài ra còn có các policy

- Permissions boundaries – json policy permission là một managed policy được sử dụng cho role và user nhằm hạn chế giới hạn phân quyền tối đa các chủ thể có thể thực hiện.Đặc điểm:


- Organizations SCPs – json policy permission Sử dụng AWS Organizations service control policy (SCP) để định nghĩa phân quyền tối đa cho các account members của một organization hoặc organization unit (ou). Tương tự Permissions boundaries không cấp quyền cho chủ thể. Đặc điểm:
  - Không ảnh hưởng tới các principal ngoài account trong resource-based policy
  - Không hạn chế quyền của user và role trong management account
  - SCP áp dụng cho tất cả user, role bao gồm các root user thuộc các account được thêm vào aws Organizations hoặc ou
  - SCP không áp dụng cho các service-linked role

- Access control lists (ACLs) được sử dụng để cho phép thực hiện cross-account access, acl chỉ định các principals của các account khác để kiểm soát truy cập. ACL không sử dụng đinh dạng JSON policy, acl không thể sử dụng với các principals trong cùng account.

- Session policies – json policy permission được sử dụng cho các temporary cendentials hoặc feradated user nhằm cấp quyền truy cập cho các định danh tạm thời dựa trên Identity-based policy được áp dụng. Có thể hiểu nếu một temporary cendentials không được cấp quyền bởi một session policy sẽ bị từ chối hành động hoặc nếu request được chấp thuận bởi session policy nhưng không được cấp quyền bởi Identity-based policy sẽ bị từ chối.

Kết luận: Nếu chia tách theo mục đính sử dụng ta sẽ có: Identity-based policy, Session policy, Resource-based policy là các chính sách cấp quyền. Với Permission boundaries, Organization SCPs là các chính sách hạn chế quyền truy cập.

#### Policy evaluation logic
<img width="1281" height="561" alt="image" src="https://github.com/user-attachments/assets/15e22706-317e-4cdd-94dc-fa559b8ac6f2" />

Được aws chia thành 6 bước đánh giá dựa trên phân loại policy:

Deny evaluation bất cứ Deny statement nào trong policy từ bất cư policy type được tổng hợp và đánh giá đầu tiên. Nếu Deny statement được áp dụng, request sẽ bị từ chối

Oraganizations policy Nếu principal thuộc một account được áp dụng Organization policy từ SCP, request phải tuân thủ policy từ Oraganization policy, nếu không sẽ bị từ chối . Ngay cả khi request đến từ aws account root user vẫn sẽ phải được cấp quyền từ Oraganization policy.

Resource-based policy Nếu policy từ Resource-based policy chấp thuận request, request sẽ được chấp thuận ngay lập tức.

IAM permission boundaries nếu principal được áp dụng permission boundaries và request không được cấp phép bởi các policy trong IAM permission boundaries, request sẽ bị từ chối.

session policy nếu principal là một temporary credentials nó phải tuân thủ session policy nếu không sẽ bị từ chối truy cập.

Identity-based policy nếu principal không được áp dụng resource-based policy thì nó bắt buộc phải được cấp quyền bởi Identity-based policy để được thự

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

- Permission boundary: giới hạn quyền của 1 user (overwrite attached permission). Note: không giúp Granted quyền, resource-based policy không bị giới hạn bởi permission boundary
  - Permission boundary của AWS IAM có thể sử dụng cả "Effect": "Allow" và "Effect": "Deny" trong các policy statement, nhưng về mặt thực tiễn và best practice, permission boundary chủ yếu dùng "Allow". Tuy nhiên, bạn hoàn toàn có thể sử dụng "Deny" trong permission boundary — nó sẽ làm giới hạn tối đa quyền mà entity có thể nhận vì deny luôn có ưu tiên cao nhất trong logic đánh giá IAM policy của AWS
  - Theo tài liệu AWS và kinh nghiệm triển khai, permission boundary nên dùng "Allow" để định nghĩa phạm vi tối đa mà entity được phép, còn các deny nên đặt ở IAM policy hoặc service control policy cho dễ kiểm soát, duy trì và audit.
  - Permission boundary giúp hạn chế việc escalate privileges (VD về leo thang quyền: 1 IAM user chỉ cần có quyền IAMFullAccess có thể tạo 1 IAM user có quyền AdmnistratorAccess để sử dụng tất cả các resources trong AWS account)


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

---

## Tổng quan phân quyền trên AWS

Phân quyền là việc tạo ra 1 tập hợp các chính sách quyết định principal (chủ thể) được phép thực thi hành động trên 1 resource hay không

Principal là đối tượng đại diện cho entity thực hiện hành động trên AWS, có thể là account, IAM user, federated user, IAM role, application. Group không được tính là principal, vì group chỉ dùng để gắn policy một cách tập trung nhưng không thể trực tiếp thực hiện hành động


Một policy được định nghĩa theo cấu trúc json và bao gồm các thuộc tính cơ bản:

<img width="657" height="402" alt="image" src="https://github.com/user-attachments/assets/fd8939f3-5f3d-4ac5-a4e4-63d737fe3976" />

Statement: Mỗi một policy đều có ít nhất một statement, statement dùng để chỉ định action nào được thực thi và tài nguyên nào được truy cập. Statement gồm các thành phần
- Sid: Là một chuỗi uniq dùng để nhận dạng statement
- Effect: Chỉ định các actions được liệt kê là Alllow/Deny
- Action: Liệt kê các hành động bạn muốn thực thi (ec2:CreateImage, ec2:CreateNetworkAcl...)
- Principal: Là tài khoản/người dùng/role được cho phép hoặc bị từ chối truy cập vào tài nguyên AWS (chỉ có khi dùng Resource-base policy)
- Resource: Chính là tài nguyên AWS mà bạn muốn áp dụng những actions bên trên
- Condition: Chỉ định các điều kiện bắt buộc phải tuân theo khi áp dụng policy này dựa trên các condition key được định nghĩa theo từng resouce, action (service-specific condition key ) và dựa trên các contex key (global condition key ).

Điểm khác biệt giữa IAM role và IAM user
- IAM User đại diện cho một thực thể cố định (ví dụ: nhân viên hoặc ứng dụng), có thông tin xác thực dài hạn như username, mật khẩu hoặc access key để truy cập AWS. Mỗi user là duy nhất và khi log in vào console hay sử dụng CLI, user sẽ sử dụng credential gắn với chính mình. Quyền hạn của IAM User được kiểm soát qua IAM Policy gắn trực tiếp hoặc qua Group.​

- IAM Role không thuộc về bất kỳ ai cố định, không có username, mật khẩu hoặc access key dài hạn. Role được “assume” khi cần – tức là có thể được gán quyền cho user, service hoặc tài nguyên AWS khác, thường dùng cho việc ủy quyền truy cập tạm thời, hoặc trao đổi quyền truy cập giữa các service/tài khoản. Role gắn với policy và chỉ cấp credential tạm thời khi được assume. Chính sách trust policy sẽ quyết định ai/what được phép assume role.

Khi một entity (user, service hoặc tài khoản khác) thực hiện “assume role”, AWS STS sẽ xác thực entity đó và cấp một bộ credential tạm thời bao gồm Access Key ID, Secret Access Key, cùng với Session Token có thời hạn ngắn (thường từ vài phút đến tối đa 12h). Những credential này cho phép entity thao tác với AWS resources như các credential dài hạn của IAM User, nhưng sẽ tự động hết hạn sau thời gian quy định và phải được xin cấp lại nếu muốn tiếp tục truy cập.

| Tiêu chí           | IAM User                                  | IAM Role                                              |
|--------------------|-------------------------------------------|-------------------------------------------------------|
| Xác thực           | Dài hạn (username/password/access key)    | Credential tạm thời, chỉ phát sinh khi assume         |
| Sở hữu             | Đại diện cho 1 người hoặc app cụ thể      | Không thuộc về ai cụ thể, ai được trust thì dùng      |
| Cách sử dụng       | Truy cập trực tiếp bằng credential        | Được assume bởi user/service/tài khoản khác           |
| Trường hợp sử dụng | Quản trị viên, app truy cập lâu dài       | Ủy quyền tạm thời, giữa accounts, federated access    |

## Giải thích IAM condition

IAM policy condition trong AWS là phần cho phép bạn chỉ định các điều kiện bổ sung để kiểm soát quyền truy cập, giúp chi tiết hóa và giới hạn quyền cho phép một cách linh hoạt hơn.

Condition nằm trong khối Statement của policy JSON, là tùy chọn.

Cấu trúc cơ bản của condition

```
 "Condition": {
    "{condition-operator}": {
        "{condition-key}": "{condition-value}"
        }
      }
```
Trong đó
- condition-operator: toán tử điều kiện (như StringEquals, NumericLessThanEquals...)
- condition-key: khóa ngữ cảnh (để so sánh)
- condition-value: giá trị để so sánh với khóa ngữ cảnh


Các loại condition phổ biến

a. aws:SourceIp : Giới hạn truy cập chỉ từ một dải IP cụ thể:

```
"Condition": {
  "IpAddress": { "aws:SourceIp": "192.0.2.0/24" }
}
```
-> Chỉ cho phép truy cập từ mạng nội bộ hoặc VPN của công ty.​

b. aws:SecureTransport : Yêu cầu truy cập phải qua HTTPS (bảo mật):

```
"Condition": {
  "Bool": { "aws:SecureTransport": "true" }
}
```
-> Dùng để bắt buộc truy cập thông qua kênh bảo mật, ngăn HTTP không mã hóa.​

c. aws:MultiFactorAuthPresent : Chỉ cho phép khi người dùng đã xác thực đa yếu tố (MFA):

```
"Condition": {
  "Bool": { "aws:MultiFactorAuthPresent": "true" }
}
```
-> Tăng cường bảo mật khi thực hiện các thao tác nhạy cảm.​

d. aws:RequestedRegion : Chỉ cho phép thao tác ở một số region cụ thể:

```
"Condition": {
  "StringEquals": { "aws:RequestedRegion": ["ap-southeast-1", "us-west-2"] }
}
```

d. aws:PrincipalTag : Giới quyền truy cập dựa trên tag

```
"Condition": {
  "StringEquals": { "aws:PrincipalTag/job-category": "iamuser-admin" }
}
```
