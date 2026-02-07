Tạo S3 bucket private, enable website hosting với index/error documents, và block public access để bảo mật.
```
resource "aws_s3_bucket" "site_bucket" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_website_configuration" "site_bucket" {
  bucket = aws_s3_bucket.site_bucket.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

resource "aws_s3_bucket_public_access_block" "site_bucket" {
  bucket = aws_s3_bucket.site_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_acl" "site_bucket" {
  depends_on  = [aws_s3_bucket_ownership_controls.site_bucket]
  bucket      = aws_s3_bucket.site_bucket.id
  acl         = "private"
}
```

Tạo S3 bucket policy cho phép cloudfront truy cập
```
resource "aws_s3_bucket_policy" "site_bucket_policy" {
  bucket = aws_s3_bucket.site_bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowCloudFrontAccess"
        Effect    = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.site_bucket.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.site_distribution.arn
          }
        }
      }
    ]
  })
}
```

Tạo OAC để CloudFront truy cập S3 private bucket một cách an toàn, thay thế OAI cũ. 
- OAC là cách CloudFront chứng minh “tôi là CloudFront”. CloudFront dùng OAC để sign request (SigV4) gửi đến S3. AWS kiểm tra rằng request đó đúng là đến từ CloudFront distribution có OAC đó, chứ không phải từ user hay tool truy cập trực tiếp S3. Còn Bucket policy là quy định ai được làm gì trên bucket. Dùng bucket policy để nói: “Cho phép service principal cloudfront.amazonaws.com đọc objects chỉ khi request đến từ CloudFront distribution có ARN này.” Nếu không có OAC: Bucket policy dù có cho phép CloudFront, nhưng CloudFront không có cơ chế sign request → S3 không nhận diện được CloudFront → access denied.
- Một OAC có thể associate với nhiều distribution (tối đa 100 distribution). Nếu bạn có hơn 100 distribution muốn trỏ về S3 bucket (hoặc nhóm bucket khác nhau), thì cần tạo thêm OAC mới để phân nhóm. Nếu bạn muốn phân biệt chính sách, signing behavior (ví dụ: phân môi trường dev/prod hoặc miền khác nhau), thì có thể tách thành nhiều OAC để dễ quản lý.

```
resource "aws_cloudfront_origin_access_control" "site_oac" {
  name                              = "oac-${var.bucket_name}"
  description                       = "OAC for S3 bucket ${var.bucket_name}"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}
```

Tạo CloudFront Distribution với S3 origin (sử dụng REST endpoint, không website endpoint), OAC, managed cache policy, HTTPS với default certificate, và compression.
​
```
resource "aws_cloudfront_distribution" "site_distribution" {
  origin {
    domain_name              = aws_s3_bucket.site_bucket.bucket_regional_domain_name
    origin_id                = "S3-${var.bucket_name}" #là một string dùng làm định danh duy nhất cho origin trong distribution. Mỗi origin trong cùng một CloudFront distribution phải có origin_id khác nhau, có thể đặt tùy ý nhưng nên đặt rõ ràng kiểu: S3-my_bucket, ALB-my_service, API-my_backend để dễ đọc code Terraform và debug
    origin_access_control_id = aws_cloudfront_origin_access_control.site_oac.id
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  aliases             = ["${var.frontend_subdomain}.${var.domain_name}"] #chỉ định các tên miền tùy chỉnh (alternate domain names hoặc CNAMEs) cho CloudFront distribution. cho phép bạn sử dụng domain riêng như www.example.com thay vì domain mặc định của CloudFront (ví dụ: d111111abcdef8.cloudfront.net). Điều này giúp route traffic từ domain tùy chỉnh đến distribution. Khai báo dưới dạng list string: aliases = ["example.com", "www.example.com"].

Phải kèm certificate hợp lệ (từ ACM hoặc CA khác) bao phủ các domain này trong khối default_cache_behavior hoặc viewer_certificate.
khi bạn khai báo aliases trong Terraform cho aws_cloudfront_distribution, CloudFront không tự tạo bản ghi CNAME hay ALIAS trong Route 53, bạn phải tự tạo DNS record

Khai báo alternate domain chỉ để cho CloudFront validate và ghi nhận các domain đấy, khi có request đến Cloudfront sẽ match Host header từ request để route đúng distribution.

chỉ tạo bản ghi DNS trong Route 53 mà không khai báo aliases sẽ KHÔNG hoạt động đúng.

CloudFront sử dụng Host header matching để xác định distribution xử lý request. Nếu www.example.com không có trong aliases list:

text
GET /index.html HTTP/1.1
Host: www.example.com  ← Không match → 403 Forbidden hoặc wrong distribution


Điều gì xảy ra nếu thiếu aliases
DNS resolution OK: Route 53 trả về IP của CloudFront edge location
​

CloudFront reject: Edge location kiểm tra Host header → không tìm thấy matching alias → 403 Forbidden
​

Certificate mismatch: SSL/TLS cert không validate cho domain không được khai báo


Tóm tắt: DNS chỉ route traffic đến edge → aliases quyết định CloudFront có chấp nhận xử lý hay không.

​



  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${var.bucket_name}"
    compress               = true
    viewer_protocol_policy = "redirect-to-https"

    cache_policy_id = "658327ea-f89d-4fab-a63d-7e22939e7934"  # Managed-CachingOptimized
  }

  price_class = "PriceClass_100"  # North America/Europe (tiết kiệm chi phí)

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
  depends_on = [aws_s3_bucket_policy.site_bucket_policy]
}
```


Outputs trả về CloudFront domain và S3 website endpoint để test.
```
output "cloudfront_domain" {
  value = aws_cloudfront_distribution.site_distribution.domain_name
}

output "s3_website_endpoint" {
  value = aws_s3_bucket_website_configuration.site_bucket.website_endpoint
}
```



