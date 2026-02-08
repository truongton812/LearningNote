Cách tạo chứng chỉ SSL/TLS trong AWS Certificate Manager (ACM) bằng Console
- Truy cập AWS Management Console tại https://console.aws.amazon.com/acm/home, chọn Request a certificate.
​- Nhập domain name (ví dụ: example.com hoặc *.example.com cho wildcard), thêm SAN nếu cần, chọn validation method (DNS hoặc email), key algorithm (RSA_2048 mặc định), và tags tùy chọn.
​- Bấm "Request" -> certificate sẽ ở trạng thái Pending validation
- Với DNS validation method, thêm CNAME record từ ACM vào DNS provider, ACM sẽ tự validate trong 72 giờ và chuyển sang Issued nếu thành công.

Cách tạo chứng chỉ SSL/TLS trong AWS Certificate Manager (ACM) bằng Terraform
- Sử dụng resource `aws_acm_certificate` để request certificate với DNS validation
```
provider "aws" {
  region = "us-east-1"  # ACM cho CloudFront phải us-east-1
}

resource "aws_acm_certificate" "example" {
  domain_name               = "example.com"
  validation_method         = "DNS"
  subject_alternative_names = ["*.example.com"]

  lifecycle {
    create_before_destroy = true
  }
}
```
- Tạo record CNAME trong Route53. Việc tạo record trong DNS provider là để AWS xác nhận bạn sở hữu domain, nếu đúng thì ACM thay đổi status từ "Pending validation" sang "Issued".
```
resource "aws_route53_record" "validation" {
  for_each = {
    for dvo in aws_acm_certificate.example.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.example.zone_id
}
```
- Tạo resource `aws_acm_certificate_validation` để chờ cho đến khi chứng chỉ đổi sang trạng thái "Issued", trước khi các resource khác phụ thuộc vào nó có thể sử dụng. Nếu không dùng resource này, certificate có thể vẫn pending khi terraform apply hoàn thành, gây lỗi cho CloudFront/ALB. Resource này đảm bảo toàn bộ stack deploy đúng thứ tự với depends_on ngầm.
```
resource "aws_acm_certificate_validation" "example" {
  certificate_arn         = aws_acm_certificate.example.arn
}
```
​
---

SAN là viết tắt của Subject Alternative Name, là trường trong chứng chỉ SSL/TLS cho phép một chứng chỉ bảo vệ nhiều tên miền/tên host ngoài domain chính (ví dụ: example.com, www.example.com, api.example.com, v.v.). Phù hợp khi bạn sở hữu nhiều domain và muốn tiết kiệm cert (nhưng không khuyến khích mix quá nhiều domain khác biệt cho quản lý).

Trong AWS ACM (và hầu hết CA khác), CN (Common Name) và SAN có thể thuộc domain khác nhau (khác base domain, khác registrar, khác hosted zone).

Ví dụ: domain_name = "example.com" (CN), subject_alternative_names = ["www.devops.net"] (SAN) - ACM sẽ chấp nhận request.

Lưu ý thực tế: Tối đa 10 SAN mỗi cert (quota mặc định).
​

