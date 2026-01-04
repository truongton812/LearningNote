Có 2 cách phổ biến để tạo “common tag” dùng chung trong Terraform:

- Dùng biến/map chung ở root module rồi truyền xuống các resource/module.

- Dùng module riêng cho common tags và merge khi cần thêm tag riêng.


C1: Dùng biến/map chung ở root module rồi truyền xuống các resource/module.

Ví dụ trong root module (main project), tạo một biến map cho tags chung:

```
variable "common_tags" {
  description = "Common tags áp dụng cho tất cả resource"
  type        = map(string)
  default = {
    Environment = "dev"
    Owner       = "devops"
    ManagedBy   = "terraform"
  }
}
```
Khi dùng với resource 
​
```
resource "aws_instance" "this" {
  ami           = "ami-xxxxx"
  instance_type = "t3.micro"

  # Tags chung + tag riêng
  tags = merge(
    var.common_tags,
    {
      Name = "app-server"
      Role = "web"
    }
  )
}
```

var.common_tags: map các tag dùng chung.

merge(...): gom common tags + tags riêng cho resource, tag trùng key ở map sau sẽ override map trước.
​

C2: Dùng module riêng cho common tags và merge khi cần thêm tag riêng.

Trong module con, expose một biến tags/labels và merge tương tự.
​

Ví dụ trong module modules/network:

```
# modules/network/variables.tf
variable "tags" {
  description = "Tags áp dụng cho resource trong module"
  type        = map(string)
  default     = {}
}
```

```
# modules/network/main.tf
resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"

  tags = merge(
    var.tags,
    {
      Name = "vpc-main"
    }
  )
}
```

Khi gọi module ở root:

```
module "network" {
  source = "./modules/network"

  tags = merge(
    var.common_tags,
    {
      Component = "network"
    }
  )
}
```
Cách này đảm bảo:
- Common tags được định nghĩa một chỗ (root).
- Mỗi module/resource vẫn có thể thêm tag riêng bằng merge.
​

---


### Giải thích rõ hơn về Default Tags trong AWS Provider
Default tags là tính năng của AWS Terraform provider (từ v3.38.0+) cho phép định nghĩa tags chung ở provider block, tự động áp dụng cho tất cả resource mà provider quản lý (trừ Auto Scaling Group).
​

```
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "production"
      Owner       = "devops-team"
      ManagedBy   = "terraform"
    }
  }
}
```
Khi tạo resource, không cần khai báo tags nữa - chúng tự động được thêm vào tags_all.
​

Override và Merge Tags

Resource-specific tags sẽ override default tags nếu trùng key.
```
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"

  # Chỉ cần tag riêng, default tags vẫn apply
  tags = {
    Name = "web-server-01"
    # Environment ở đây sẽ override default nếu có
  }
}
```
Kết quả: instance có cả default tags + tags riêng (trùng key lấy giá trị resource).
​

Xử lý Auto Scaling Group (ASG)

Lưu ý Default tags không apply tự động cho ASG và EC2 instances do ASG tạo. Phải dùng data source + dynamic block:
​

```
data "aws_default_tags" "current" {}

resource "aws_autoscaling_group" "example" {
  # ... config khác

  dynamic "tag" {
    for_each = data.aws_default_tags.current.tags
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true  # Quan trọng: propagate xuống EC2 instances
    }
  }
}
```
Sau terraform apply -replace ASG, instances mới sẽ có full tags​
