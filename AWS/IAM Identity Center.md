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
