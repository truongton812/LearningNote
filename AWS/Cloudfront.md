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
#1 distribution có thể khai báo tối đa 100 origins
  origin {
    domain_name              = aws_s3_bucket.site_bucket.bucket_regional_domain_name
    origin_id                = "S3-${var.bucket_name}" #là một string dùng làm định danh duy nhất cho origin trong distribution. Mỗi origin trong cùng một CloudFront distribution phải có origin_id khác nhau, có thể đặt
tùy ý nhưng nên đặt rõ ràng kiểu: S3-my_bucket, ALB-my_service, API-my_backend để dễ đọc code Terraform và debug
    origin_access_control_id = aws_cloudfront_origin_access_control.site_oac.id
  }
  
# Origin 2: API từ ALB
  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "APILoadBalancer"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  aliases             = ["${var.frontend_subdomain}.${var.domain_name}"] # Khai báo dưới dạng list string. Lưu ý cần kèm certificate hợp lệ (từ ACM hoặc CA khác) bao phủ các domain này trong khối default_cache_behavior hoặc viewer_certificate.
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${var.bucket_name}"
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
    cache_policy_id = "658327ea-f89d-4fab-a63d-7e22939e7934"  # Managed-CachingOptimized
  }

  # Custom behavior: API calls
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    target_origin_id = "APILoadBalancer" 
    allowed_methods  = ["GET", "POST", "PUT", "DELETE"]
    # TTL ngắn hoặc no-cache cho API
    min_ttl     = 0
    default_ttl = 60
    max_ttl     = 300
    forwarded_values {
      query_string = true         # Forward tất cả query params cho API
      cookies { forward = "all" }
    }
  }
  ordered_cache_behavior {
    path_pattern     = "/images/*"
    target_origin_id = "ImageBucket"
    min_ttl          = 86400  # Cache hình ảnh 1 ngày
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

---

`Alternate domain names` dùng để chỉ định các custom domain (VD www.example.com) cho CloudFront distribution thay vì domain mặc định của CloudFront (VD d111111abcdef8.cloudfront.net). 

Mục đích của `Alternate domain names` là để whitelist domain mà Cloudfront chấp nhận xử lý. Khi request đi đến Cloudfront, Edge location sẽ kiểm tra trường Host header trong request, nếu không khớp với `Alternate domain names` sẽ reject. Điều này đảm bảo không phải ai cũng có thể trỏ DNS đến distribution

Lưu ý user cần phải tự tạo DNS record (CNAME/Alias) trong DNS 

---

Cache Behavior trong CloudFront là quy tắc định nghĩa cách xử lý và cache request cho các path pattern cụ thể.

Cache Behavior quyết định:
- Origin nào phục vụ content (S3, EC2, ALB...)
- Cache bao lâu (TTL settings)
- Forward gì đến origin (query string, headers, cookies)

Default cache behavior (*): Áp dụng cho tất cả request không match custom behavior. Còn Custom behavior sẽ xử lý riêng cho từng path (VD /images/*, /api/*, *.css). Thứ tự ưu tiên Cache Behavior được quyết định bằng vị trí trong ordered_cache_behavior list (từ trên xuống)


Các trường chính trong khai báo Cache behavior
- target_origin_id (required): ID của origin sẽ phục vụ request này
- path_pattern (dùng cho ordered_cache_behavior): VD pattern như `/api/*`, `/images/*.jpg`
- viewer_protocol_policy:	(required): chỉ định protocol mà users có thể dùng để truy cập file trong origin. Có thể chọn các giá trị `allow-all`, `https-only` hoặc `redirect-to-https`
- allowed_methods:	HTTP methods cho phép: `["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]`
- Cache TTL Settings (deprecated, khuyến nghị dùng Cache Policy)
- forwarded_values (deprecated, khuyến nghị dùng Policy)
