<img width="1403" height="442" alt="image" src="https://github.com/user-attachments/assets/96b4c7cc-aa57-49b5-8534-475192c38754" />

Đây là giao diện AWS access portal, nơi bạn đăng nhập bằng một danh tính (identity) do IAM Identity Center quản lý để truy cập các tài khoản AWS và thực hiện switch vai trò (role).​

Giải thích các thành phần:

Account: Là dòng chứa tên như "Audit-Bccloud-Test" hoặc "BCCloud (test)"; đây là tài khoản AWS thực tế trong tổ chức của bạn, được định danh bằng account ID và email liên kết (ví dụ: 866982151093).​

Role: Các dòng như "G.U.Infra-Admin" hoặc "G.U.Security" xuất hiện bên dưới mỗi account; đây là các permission set đã được gán cho tài khoản ( "tài khoản" ở đây chính là từng AWS account nằm trong tổ chức AWS Organizations của bạn). Khi bạn gán permission set (ví dụ: "G.U.Infra-Admin" hoặc "G.U.Security"), permission set này sẽ được liên kết với một hoặc nhiều IAM role được tự động tạo ra trong từng AWS account đó. Danh tính (identity - user/group trong IAM Identity Center) có thể assume vào các role đó sau khi đăng nhập portal, để truy cập tài nguyên trong tài khoản AWS tương ứng

Identity: Chính là tài khoản bạn dùng để đăng nhập vào portal này. Identity ở đây là user/group của IAM Identity Center, không phải là IAM user hay IAM role riêng lẻ trong từng account. Một identity (user hoặc group) sẽ nhìn thấy các account và vai trò/permission set mà tổ chức đã phân quyền cho họ. Một identity trong IAM Identity Center sẽ tương đương với một tên đăng nhập (username) vào AWS portal. Mỗi user (hoặc danh tính) được tạo hoặc đồng bộ vào IAM Identity Center đều có thể đăng nhập vào cổng truy cập AWS (access portal) bằng username/email và mật khẩu riêng. Khi đăng nhập, người dùng sẽ thấy các tài khoản AWS và quyền (permission set) được cấp theo phân quyền quản trị trung tâm


<img width="776" height="420" alt="image" src="https://github.com/user-attachments/assets/9eb5c5e0-6a89-4f77-a921-1e63049e9064" />



Trong hình này, trường "Username / group name" chính là danh sách các user hoặc group (nhóm người dùng) trong IAM Identity Center đã được gán quyền truy cập vào một AWS Account thông qua các permission set cụ thể

Đây không phải là "role" của AWS theo khái niệm truyền thống (IAM role), mà là user hoặc group đại diện cho danh tính (identity) của bạn trong hệ thống quản trị tập trung (IAM Identity Center). Khi bạn gán một permission set cho user/group đó, thì IAM Identity Center sẽ tạo ra một IAM role trong tài khoản AWS tương ứng với permission set đó. Người dùng hoặc nhóm sẽ “assume” (nhận) role đó khi đăng nhập vào AWS thông qua Identity Center

Lý do "username / group name" không phải là role là vì:

- Username / group name là tên người dùng (user) hoặc tên nhóm (group) được quản lý bởi IAM Identity Center. Đây là danh tính thực tế dùng để đăng nhập vào AWS Access Portal.​

- Role (IAM role) trong AWS là một loại identity đặc biệt dùng để ủy quyền tạm thời, cho phép user, service hoặc group thực hiện các tác vụ với chính sách nhất định, nhưng nó không phải là bản thân user hay group. Khi bạn truy cập tài khoản AWS qua IAM Identity Center, hệ thống tạo IAM role tạm thời trong account và cho phép identity (user/group) "assume" (nhận) role đó.​

Chúng dễ bị nhầm lẫn vì:

- Một identity (user hoặc group) sẽ được gán một hoặc nhiều permission set, mỗi permission set sẽ triển khai thành một IAM role thực tế trong AWS account.

- Khi đăng nhập, bạn sẽ "assume" role do permission set quy định, nhưng thực chất bạn vẫn đăng nhập portal với danh tính user/group, chứ không phải role trực tiếp

---

# AWS IAM Identity Center

Là dịch vụ cung cấp SSO vào nhiều tài khoản AWS trong Organization. Lưu ý: chỉ khi dùng Organization mới có thể truy nhập đa tài khoản, nếu dùng 1 tài khoản thì chỉ login được 1 tài khoản đấy.

## Các khái niệm:

- User: Là 1 thực thể đại diện cho 1 cá nhân, được tạo trong Identity Center hoặc đồng bộ từ external Identity Provider.

- Account (tài khoản): Là AWS account. Identity Center cho phép quản trị viên gán cho user quyền truy cập vào các AWS account thông qua permission set.

- Identity Provider: là nơi lưu trữ thông tin của các user (Trong trường hợp user không được tạo trong Identity Center). IdP có thể giao tiếp với Identity Center thông qua 2 giao thức:
  - SAML 2.0: hỗ trợ Okta, Azure AD, OneLogin,…. SAML 2.0 không expose API cho AWS query để biết về user và group lưu trong eIdP → ta phải tạo thủ công user/group trong Identity Center.
  - SCIM: giao thức này tự động sync user từ IdP vào Identity Center.

- Permission set: dùng để định nghĩa quyền mà user có thể truy cập vào account. Mỗi permission set sẽ tương ứng với 1 role trong account. Khi đăng nhập vào Identity Center Portal, user sẽ thấy danh sách các tài khoản AWS kèm theo các role (tương ứng với 1 permission set) để chọn nhằm assume vào 1 role. Có thể gán nhiều permission set trên cùng 1 account → user có thể chọn 1 trong các role để đăng nhập vào 1 account.



## Cách Identity Center hoạt động

- Ta thêm các AWS account vào Organization.

- Tạo user/group trong Identity Center hoặc đồng bộ từ external Identity Provider sang.

- Tạo permission set để xác định quyền truy cập.

- Gán permission set vào account. Cách làm: chọn vào account, gán permission set và user vào → Identity Center sẽ tạo IAM role trong account đấy với quyền bằng permission set chỉ định. Lưu ý permission set chỉ giới hạn quyền của user tạo trong Identity Center, không ảnh hưởng đến user/role trong AWS account

<img width="1892" height="684" alt="image" src="https://github.com/user-attachments/assets/0d45881d-43d5-4036-8202-9bb16f0bfb8a" />

- User đăng nhập vào Identity Center Portal sẽ thấy danh sách các AWS account mà user được cấp quyền truy cập cùng với các role tương ứng. Khi user chọn 1 account và role, hệ thống sẽ cho user assume vào role đấy trong account bằng temporary credential.

