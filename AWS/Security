AWS Security Hub là dịch vụ quản lý bảo mật tập trung của AWS, giúp tổng hợp, phân tích và quản lý các cảnh báo và phát hiện bảo mật từ nhiều nguồn khác nhau. Nó cung cấp một bảng điều khiển duy nhất để theo dõi trạng thái bảo mật và tuân thủ của môi trường AWS, hỗ trợ các tiêu chuẩn bảo mật như CIS, PCI DSS và ISO 27001.

Nguồn dữ liệu của Security Hub chủ yếu lấy từ các dịch vụ bảo mật nền tảng của AWS, gồm:

- Amazon GuardDuty (phát hiện tấn công và hành vi bất thường)

- AWS Config (giám sát cấu hình tài nguyên và vi phạm)

- Amazon Macie (phát hiện rò rỉ dữ liệu nhạy cảm)

- Amazon Inspector (đánh giá điểm yếu bảo mật cho EC2 và container)

- AWS Firewall Manager

- IAM Access Analyzer

Ngoài ra, Security Hub còn tích hợp được với các sản phẩm bảo mật của bên thứ ba như CrowdStrike, Splunk, Trend Micro, giúp tập trung tất cả thông tin bảo mật vào một chỗ để dễ dàng xử lý, ưu tiên và tự động hóa phản ứng bảo mật. Tất cả dữ liệu phát hiện được chuẩn hóa theo định dạng AWS Security Finding Format (ASFF) để dễ dàng quản lý và kết hợp thông tin từ nhiều nguồn.

Tóm lại, AWS Security Hub lấy dữ liệu từ nhiều dịch vụ bảo mật AWS cốt lõi và các giải pháp đối tác, tổng hợp, chuẩn hóa và phân tích để cung cấp cái nhìn toàn diện về tình trạng an ninh và tuân thủ của hệ thống AWS, đồng thời hỗ trợ tự động hóa phản ứng bảo mật

---

Flow xử lý với hạ tầng AWS nhiều account, đặc biệt khi triển khai SIEM, SOC, hoặc các giải pháp security automation


- Các account gửi dữ liệu log, cảnh báo của riêng mình về Audit Account qua các liên kết đã cấu hình.

- Audit Account thực hiện các tác vụ xử lý, phân tích bảo mật và chuyển dữ liệu hợp nhất về Monitoring Account.

- Monitoring Account nhận dữ liệu bảo mật (Guard Duty findings, Security Hub findings) từ các account khác và dùng các dữ liệu đấy để xây dựng dashboard, hệ thống SIEM xử lý toàn diện.

```
[Monitored Account 1] ---->|
                          |--> [Audit Account] ---> [Monitoring Account]
[Monitored Account 2] ---->|
```
