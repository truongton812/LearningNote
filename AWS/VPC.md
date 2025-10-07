Mối liên hệ giữa Subnet và Availability Zone (AZ) trong AWS như sau:

Một Availability Zone (AZ) là một vị trí vật lý riêng biệt trong một AWS Region. Mỗi AZ gồm một hoặc nhiều trung tâm dữ liệu (data center) độc lập, giúp tăng tính sẵn sàng và khả năng chịu lỗi cho hệ thống.

Một Subnet là một mạng con (subnetwork) được tạo trong một VPC và nằm hoàn toàn trong một Availability Zone duy nhất. Nghĩa là, mỗi subnet chỉ gói gọn trong một AZ, không thể kéo dài qua nhiều AZ.

Khi tạo subnet trong AWS, bạn cần chỉ định CIDR block cho subnet đó và chọn một AZ cụ thể để subnet thuộc về.

Mỗi VPC có thể có nhiều subnet, mỗi subnet nằm trong một AZ khác nhau nhằm phân bố tài nguyên, hệ thống của bạn có thể được triển khai trên nhiều AZ khác nhau để tăng độ dự phòng và khả năng chịu lỗi.

Việc phân chia subnet theo AZ giúp AWS và bạn kiểm soát mạng tốt hơn, phân tách tài nguyên theo vùng vật lý, dễ dàng tổ chức kiến trúc dịch vụ phân tán, đảm bảo khi AZ này gặp sự cố thì các subnet (và tài nguyên) ở AZ khác vẫn hoạt động bình thường.

Khi tạo tài nguyên EC2 trong AWS, bạn cần chỉ định Subnet chứ không phải chỉ định trực tiếp Availability Zone (AZ).

Tóm lại:
Subnet là mạng con được chứa trong một AZ cố định, còn AZ là khu vực vật lý/phần cứng của AWS trong một Region. Mối quan hệ này giúp phân tách và cách ly tài nguyên trong mạng một cách vật lý và logic hiệu quả.
