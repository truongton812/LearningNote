CNAME: trỏ được về aws và non-aws resource. TUy nhiên chỉ work với non-root domain

Alias: chỉ trỏ được về aws resource (VD ELB, CloudFront distribution, Elastic Beanstalk, API GW, VPC endpoint, S3 website bucket). Work với cả root và non-root domain. Ưu điểm là không mất phí

Root domain là tên miền gốc, ví dụ: “example.com” (kết hợp giữa domain name và TLD).

Non-root domain là mọi biến thể hoặc phần mở rộng từ root domain, ví dụ:

- Subdomain: “blog.example.com”, “shop.example.com” – đây là các non-root domain vì chúng thêm tiền tố phía trước tên miền gốc.

- Địa chỉ có www: “www.example.com” cũng có thể gọi là non-root domain vì “www” là một subdomain mặc định, không phải tên miền gốc thực sự.


Root domain luôn là địa chỉ cơ bản nhất cho một website, ví dụ “tuna-devops.site”.

Non-root domain bao gồm bất cứ thứ gì xuất hiện trước tên miền gốc, thường là các subdomain.
