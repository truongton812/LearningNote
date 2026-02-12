Khi bạn “join” vào AWS Organization (tức là tài khoản của bạn trở thành member account trong Organization), thì các resource bạn tạo sẽ nằm trên tài khoản mà bạn đang dùng để tạo resource đó, không phụ thuộc vào việc bạn đã join Organization hay chưa.

AWS Organization chỉ là layer quản lý tập trung (policies, SCPs, consolidated billing, OUs…), chứ không “di chuyển” tài nguyên sang management account.

AWS quy định rằng tài khoản AWS là owner của các resource được tạo trong tài khoản đó, bất kể principal là root user, IAM user hay IAM role. Có nghĩa là:
- Tạo resource trong member account → ownership thuộc member account.
- Tạo resource trong management account → ownership thuộc management account.

Nếu bạn dùng IAM user / role trong member account để tạo EC2, S3, RDS… thì resource sẽ thuộc member account đó, dù account đã nằm trong Organization.

Nếu bạn dùng IAM role trong management account để assume role sang member account rồi tạo resource, thì resource vẫn nằm trên member account, nhưng hành động được thực hiện qua trust relationship từ management account.


Bạn không thể trực tiếp dùng một member account để tạo resource “thuộc” management account, nhưng bạn có thể dùng member account để thực hiện hành động tạo resource trong management account, nếu có trust relationship và quyền phù hợp.
