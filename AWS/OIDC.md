OpenID Connect (OIDC) là giao thức xác thực được xây dựng mở rộng trên OAuth 2.0, giúp ứng dụng xác minh danh tính người dùng (hoặc workload) và lấy thông tin profile cơ bản một cách chuẩn hóa, an toàn.
​

Khái niệm và thành phần chính

OIDC = OAuth 2.0 + xác thực: OAuth 2.0 lo về ủy quyền (resource nào được truy cập), OIDC bổ sung lớp xác thực để biết “ai” đang dùng ứng dụng.
​

ID Token: JWT chứa thông tin danh tính (claims) như sub (user id), email, name, audience, issuer, thời gian hết hạn… dùng để app tin rằng người dùng đã được xác thực.
​

OpenID Provider (OP): Máy chủ xác thực phát hành ID Token (ví dụ: Google, Microsoft, GitLab).
​

Relying Party (Client): Ứng dụng tin tưởng OP, nhận ID Token và dùng để login, kiểm soát session.
​

Luồng hoạt động cơ bản

Một luồng OIDC điển hình (Authorization Code Flow) diễn ra như sau:
​
- Người dùng truy cập ứng dụng (RP) và chọn “Đăng nhập với X” (Google, GitLab…).
- Ứng dụng redirect người dùng sang OpenID Provider để login và cấp quyền.
- OP xác thực người dùng (nhập username/password, MFA…).
- Sau khi thành công, OP trả về cho ứng dụng:
  - Một ID Token (JWT) – chứng minh danh tính.
  - Thường kèm Access Token để gọi API resource nếu cần.
- Ứng dụng validate ID Token (chữ ký, iss, aud, exp…) rồi tạo session cho user.

Trong trường hợp GitLab CI → AWS, GitLab đóng vai trò OP phát hành ID Token cho pipeline, AWS là bên tin cậy (RP) dùng token đó để quyết định có cho assume role hay không.
​
Cách hoạt động trong GitLab-AWS
```
GitLab CI Job → Request ID Token → GitLab (OIDC Provider) → JWT Token
                    ↓
AWS IAM → Verify JWT → Assume IAM Role → Temporary AWS Credentials (1h)\
```
GitLab CI job gửi request token đến Gitlab

GitLab (IdP) tạo JWT ID Token chứa claims về pipeline (project_path, branch, job ID) – chứng minh "pipeline này từ repo X, branch Y đang chạy".

GitLab CI gửi token này qua aws sts assume-role-with-web-identity đến AWS STS (không cần AWS credentials ban đầu).

AWS IAM verify:
- Kiểm tra chữ ký JWT từ GitLab OIDC Provider (đã add trước).
- Match conditions: aud, sub (ví dụ: chỉ cho phép repo cụ thể).
- Nếu OK → cấp temporary credentials cho IAM Role.

Nếu JWT hợp lệ → aws sts assume-role-with-web-identity nhận credentials để deploy EC2/EKS.

Lưu ý: cần tạo trust giữa AWS IAM và GitLab OIDC Provider để AWS chấp nhận token từ GitLab.
​

Cách tạo trust (2 bước chính)

1. Add OIDC Identity Provider trên AWS IAM

IAM Console > Identity providers > Add provider > OpenID Connect.


2. Tạo IAM Role với Trust Policy
```
{
  "Version": "2012-10-10",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR-ACCOUNT:oidc-provider/gitlab.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": { #Giới hạn project/branch/tag cụ thể (dùng ref_type:branch:ref:main cho main branch).
        "StringEquals": {
          "gitlab.com:aud": "https://gitlab.com"
        },
        "StringLike": {
          "gitlab.com:sub": "project_path:your-group/your-project:*"
        }
      }
    }
  ]
}
```​
