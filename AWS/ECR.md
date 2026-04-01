| Thành phần | Tên gọi |
|---|---|
| `217441068302.dkr.ecr.ap-southeast-1.amazonaws.com` | **Registry URI** |
| `217441068302.dkr.ecr.ap-southeast-1.amazonaws.com/geneco-staging-ops` | **Repository URI** |
| `217441068302.dkr.ecr.ap-southeast-1.amazonaws.com/geneco-staging-ops:a0c6bbe3...` | **Image URI** |
| `geneco-staging-ops` | **Repository name** |
| `a0c6bbe3499c5e40a13e8ecdb3bae0718b270d37` | **Image tag** |

Trong đó

URI (Uniform Resource Identifier) là khái niệm rộng hơn — chỉ dùng để định danh một resource, không nhất thiết phải truy cập được.

URL (Uniform Resource Locator) là subset của URI — vừa định danh vừa cho biết cách truy cập (protocol + location).

Nói đơn giản: mọi URL đều là URI, nhưng không phải URI nào cũng là URL.

Ví dụ:
- `https://example.com/page` là URL (có protocol https, truy cập được qua web) và đồng thời cũng là URI
- `arn:aws:s3:::my-bucket` là URI (định danh resource trên AWS), không phải URL (không có protocol để "truy cập")
- `217441068302.dkr.ecr...` là URI (định danh image trong ECR), không phải URL chuẩn
