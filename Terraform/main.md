
# Terraform

## 1. General
## 1.1 Terraform phase

Terraform hoạt động qua các phases:
- init: initilize project và identify provider. Khi chạy lệnh init thì Terraform sẽ download và cài đặt plugin cho provider trong file .tf. Plugin sẽ được download về trong file .Terraform/plugin (nằm trong cùng thư mục chứa file .tf)
- plan: xác định và hiển thị những thay đổi nào sẽ được thực hiện đối với hạ tầng trước khi các thay đổi thực sự được áp dụn
- apply: thực thi thay đổi trên môi trường thực tế hoặc đưa môi trường thực tế về đúng desired state nếu có thay đổi. Dùng option  -auto-approve để bỏ qua xác nhận
- destroy: dùng để xóa resources

### 1.2 Các lệnh làm việc với Terraform
- `Terraform validate` : dùng để kiểm tra syntax
- `Terraform fmt` : format lại Terraform configuration file cho dễ đọc
- `Terraform show` : print current state của infrastructure . Thêm option -json để đọc dưới dạng json
- `Terraform providers` : xem tất cả providers có trong thư mục hiện tại
- `Terraform providers mirror /path/to/other/configuration/directory` : copy provider ở current directory sang directory khác
- `Terraform output` : in ra tất cả output trong thư mục configuration directory hiện tại. Thêm tên của output variable để chỉ lấy output của variable đấy
- `Terraform plan/apply -target=<resource_address>` : plan/apply thay đổi chỉ cho một hoặc một số resource cụ thể được xác định bởi resource address (VD aws_instance.mytestinstance)
- `Terraform apply -refresh-only` : dùng để sync file state với real-world state. Dùng trong trường hợp có thay đổi manually trên real-world infrastructure thì chạy lệnh này để update state file. Lưu ý là lệnh này chỉ modify state file
- option `-resfresh=false` để bypass việc refresh Terraform state
- `Terraform destroy` : dùng để xóa tất cả resource. Tuy nhiên best practice là comment các resource muốn xóa và apply lại
- `Terraform graph` : visualize ra mối quan hệ dependencies giữa các resource (cần phải cài phần mềm đọc format dot)

## 2. Hashicorp Configuration Language

Syntax:
```
<block> <parameters> {
  key1 = value1
  key2 = value2
}
```

Trong đó block chứa thông tin về infrastruture platform và các resources ta muốn tạo

Các giá trị có thể nhận trong block là Provider/Resource/Variable/Output/Module


## 3. Các loại file trong Terraform và công dụng

Trong thư mục Terraform có thể có các file
- *main.tf* : chứa tất cả resource cần tạo
- *variables.tf* : Khai báo variable                         |
- *outputs.tf* : chứa output từ resources                       
- *provider.tf* : chứa thông tin provider (VD AWS, Azure, GCP,...) và credential để kết nối đến provider                        

### 3.1. Provider.tf
- Dùng để khai báo và cấu hình Provider — tức là nhà cung cấp dịch vụ cloud hoặc hạ tầng mà bạn làm việc (ví dụ AWS, Azure, Google Cloud).
- Có thể khai báo nhiều provider trong cùng 1 file -> khi chạy Terraform init thì sẽ download tất cả các provider plugin được khai báo  (check trong directory `.Terraform`)
- Với dự án nhỏ, đơn giản có thể đưa provider vào main.tf cho tiện. Tuy nhiên với dự án lớn, đa môi trường, hay nhóm nhiều người làm, nên tách riêng provider.tf để giữ cấu trúc code sạch, rõ ràng, dễ maintain và dễ triển khai automation.
- Cấu trúc file provider
```
provider "aws" {
  profile = "my-user" #optional, nếu không khai báo terraform sẽ sử dụng profile default
  region  = "us-east-2"
}

terraform { #khai báo constrain version cho provider
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.94"
    }
  }
}
```

### 3.2. Main.tf
- Dùng để khai báo resources cần tạo
- Ví dụ để tạo EC2 trên AWS

```
provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
}
```
#### 3.2.1. Refer attribute của resource
- Dùng khi muốn lấy attribute của resource này để làm input cho resource khác. VD lấy VPC ID để đưa vào tạo EC2 instance
- Syntax: `resource_type.resouce_name.attribute`
- Khi định nghĩa resource reference như vậy thì mặc định đã implicitly chỉ định thứ tự tạo resource cho Terraform. Lưu ý ngoài cách chỉ định implicitly còn có thể chỉ định explicitly dependencies bằng cú pháp `depends_on = [ resource_type.resource_name ]` (depends_on là 1 list nên có thể chỉ định nhiều items)
- Ví dụ
```
resource "aws_instance" "example" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"
}

resource "aws_eip" "example_eip" {
  instance = aws_instance.example.id
  vpc      = true
}
```
### 3.3. Variables.tf

Dùng để khai báo variable

#### 3.3.1 Cách sử dụng

- Syntax khai báo: `variable "content" {}`. Example:

```
variable "filename" {
  default = "/root/pets.txt" #optional
  type = string #optional. Ngoài ra có thể nhận giá trị number, bool, list, map, object, tuple, set. Nếu không define thì default sẽ là any (tức có thể là bất kỳ type nào)
  description = "the file name" #optional
}
```

- Syntax để gọi variable `var.<variable_name>`. Example:

```
resource "local_file" "pet" {
  filename = var.filename
  content  = var.content
}
```

- Nếu trong file variable.tf không khai báo giá trị default cho variable thì có các cách sau để truyền giá trị cho variable: (thứ tự ưu tiên tăng dần)
  - Truyền variable bằng cách export vào biến môi trường (luôn phải có tiền tố TF_VAR ở trước). VD `export TF_VAR_filename="root/pet.txt"`
  - Đặt biến trong file Terraform.tfvars hoặc Terraform.tfvars.json (hoặc file khác chỉ cần có đuôi là *auto.tfvars hoặc *auto.tfvars.json ). Biến được khai bao với định dạng `key = value`. Khai báo biến trong Terraform.tfvars giúp việc sửa biến rõ ràng hơn. Nếu đặt tên file khác các tên trên thì phải chỉ định file chứa variable bằng option `Terraform apply -var-file <ten_file>`
  - Truyền variable bằng option -var. VD `Terraform apply -var "filename=/root/pet.txt" -var "content=We love pet"`
  - Truyền khi chạy lệnh `Terraform apply` sẽ có prompt để nhập giá trị cho variable.
 
- Ngoài `variable` còn có `local value`. Local value là các biến chỉ có giá trị nội bộ trong module nơi được khai báo, không thể truy cập từ module khác. VD nếu  khai báo locals ở module gốc (root), chỉ các resource, data, output trong root module dùng được. Module con (child) sẽ không thể truy xuất local của root.​ Hoặc nếu khai báo locals ở module con, chỉ các resource, output bên trong module con đó mới truy cập được. Local value không nhận giá trị bên ngoài (VD qua CLI, file, ENV) như variable
- Syntax khai báo `local value`
```
locals {
  env     = "staging"
  region  = "us-east-2"
  zone1   = "us-east-2a"
  zone2   = "us-east-2b"
  eks_name = "demo"
}
```
- Syntax để gọi local value: `local.value` . VD: `{ Name = "${local.env}-main" }` (Name tag là tag đặc biệt trong AWS, giúp hiển thị tên resource lên giao diện console)

#### 3.3.2 Variable type

##### 3.3.2.1 List
- Là một danh sách có thứ tự các phần tử có cùng kiểu dữ liệu, cho phép trùng lặp giá trị
- VD khai báo 1 list:

```
variable "prefix" {
  type = list #ngoài ra có thể tạo constraint như type = list(number) -> tất cả các phần tử của list phải là number, nếu không Terraform sẽ báo lỗi
  default = ["Mr", "Mrs", "Sir]
}
```
- Gọi từng phần tử của list:
```
resource "random_pet" "mypet" {
  prefix = var.prefix[0]
```
- Lấy danh sách các phần tử của list:
```
locals {
  upper_list = [for item in var.example_list : upper(item)]
}
```
- Duyệt các phần tử của list bằng vòng lặp count hoặc for-each
```
resource "aws_security_group" "example" {
  count = length(var.ip_ranges)

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [var.ip_ranges[count.index]]
  }
}
```

##### 3.3.2.2. Set: 
- Là một danh sách có thứ tự các phần tử có cùng kiểu dữ liệu, nhưng không cho phép trùng lặp giá trị nếu không sẽ báo lỗi
- VD khai báo 1 set:
```
variable "prefix" {
  type = set #ngoài ra có thể tạo constraint như type = set(string) -> tất cả các phần tử của set phải là string, nếu không Terraform sẽ báo lỗi
  default = ["Mr", "Mrs", "Sir]
}
```
##### 3.3.2.3. Map
- Map là một cấu trúc dữ liệu key-value, trong đó tất cả các value phải có cùng kiểu dữ liệu, còn key trong map phải là kiểu string. Ví dụ map(string) có nghĩa các value đều là chuỗi. Lưu ý key là duy nhất trong cùng một map, bạn không thể có 2 key giống nhau
- Trong Terraform, một map có thể được chuyển đổi thành object nếu nó có ít nhất các key phù hợp với schema của object, nhưng có thể mất các thuộc tính dư thừa. Việc chuyển đổi ngược lại không tự động.
- VD khai báo 1 map:
```
variable "example_map" {
  type = map #ngoài ra có thể tạo constraint như type = map(string) -> tất cả các value phải là string, nếu không Terraform sẽ báo lỗi
  default = {
    "dev"     = "192.168.1.1"
    "staging" = "192.168.1.2"
    "prod"    = "192.168.1.3"
  }
}
```
- Gọi các phần tử của map:
```
resource local_file my-pet {
  content = var.example_map["dev"]
```
- Lấy danh sách các giá trị từ map:
```
locals {
  ip_list = [for key, val in var.example_map : val]
}
```
- Duyệt từng cặp key-value để xử lý:
```
resource "example_resource" "r" {
  for_each = var.example_map

  name = each.key
  ip   = each.value
}
```

##### 3.3.2.4. Object
- Một object định nghĩa một schema hoặc cấu trúc rõ ràng với các thuộc tính được đặt tên (named attributes), mỗi thuộc tính có thể có kiểu dữ liệu khác nhau. Thuộc tính của object có thể thuộc các kiểu: string, number, bool, list, set, map, object, tuple. Ví dụ một object có thể có thuộc tính name kiểu string, age kiểu number, favorite_pet kiểu bool. Object dùng để định nghĩa kiểu dữ liệu phức tạp với nhiều trường riêng biệt, mỗi trường có kiểu riêng.
- Có thể hiểu đơn giản là map là một tập hợp các giá trị cùng kiểu với key tùy ý, còn object là một kiểu dữ liệu có cấu trúc cố định với các trường đã xác định sẵn kiểu của từng trường.
- Ví dụ khai báo 1 object
```
variable "bella" {
  type = object({
    name         = string
    color        = string
    age          = number
    food         = list(string)
    favorite_pet = bool
  })

  default = {
    name         = "bella"
    color        = "brown"
    age          = 7
    food         = ["fish", "chicken", "turkey"]
    favorite_pet = true
  }
}
```

##### 3.3.2.5. Tuple
- Tuple: Tuple là một tập hợp có số lượng phần tử cố định, thứ tự cố định và có thể chứa các phần tử với kiểu dữ liệu khác nhau. Tuple là bất biến (immutable), nghĩa là không thể thay đổi, thêm hoặc xóa phần tử sau khi đã tạo.
- Tuple khác với list là list là một tập hợp phần tử có thứ tự, có thể thay đổi kích thước, thường chứa các phần tử cùng kiểu dữ liệu và có thể thêm, sửa, hoặc xóa phần tử trong danh sách (mutable).
- Ví dụ khai báo 1 tuple:
```
variable kitty {
  type = tuple #ngoài ra có thể tạo constraint như type = tuple([string, number, bool])
  default = ["cat",7,true]
```

### 3.4. Outputs.tf
- Là biến được sử dụng để export các giá trị từ trong Terraform state sau khi thực thi. Output variable cho phép lấy các thông tin của tài nguyên vừa tạo (ví dụ: địa chỉ IP của EC2, ID của VPC, URL của Load Balancer) và hiển thị chúng ra terminal hoặc truyền cho các thành phần khác sử dụng.
-  Công dụng của output variable
  - Giúp tra cứu nhanh các thông tin của tài nguyên sau khi Terraform apply.
  - Cho phép các module con trả về dữ liệu cho module gọi (parent module).
  - Hữu ích khi tích hợp Terraform trong CI/CD pipeline, để các bước kế tiếp lấy dữ liệu dynamic.
- Output có thể khai được trong bất kỳ file .tf nào, ví dụ trong main.tf, tuy nhiên best practice là nên tách biệt output ra file riêng để codebase chuyên nghiệp và dễ maintain hơn. Việc để output trong file outputs.tf giúp:
  - Giữ cấu trúc code sạch sẽ, dễ quản lý.
  - Dễ dàng kiểm soát các giá trị đầu ra của toàn bộ dự án hoặc module.
  - Người khác khi làm việc trong dự án dễ dàng tìm thấy phần output riêng biệt để tham chiếu hoặc sử dụng.
- Ví dụ sử dụng output
```
output "instance_public_ip" {
  description = "Public IP của instance EC2"
  value       = aws_instance.example.public_ip
}
```
- Khi dùng lệnh `Terraform output` sẽ in ra tất cả output của configuration file trong thư mục hiện tại hoặc lệnh `Terraform output <output_name>` để in ra specific output

## 4. Cách tổ chức thư mục Terraform cho nhiều môi trường
### 4.1 Scnerio 1: Các môi trường có cùng resources

Cách tổ chức thư mục

```
.
├── provider.tf   
├── main.tf               # Đặt ở root, chứa resource đồng nhất cho mọi môi trường
├── variables.tf          # Khai báo các biến dùng chung
├── outputs.tf            # (Nếu có) Xuất ra các giá trị cần thiết
├── env/
│   ├── dev/
│   │   └── terraform.tfvars   # Giá trị biến cho môi trường dev
│   ├── prod/
│   │   └── terraform.tfvars   # Giá trị biến cho môi trường prod
│   └── staging/
│       └── terraform.tfvars   # Giá trị biến cho môi trường staging (nếu cần)
```

- Khai báo tất cả resource, provider, biến dùng chung trong các file ở thư mục root (main.tf, variables.tf, v.v).
- Trong mỗi thư mục môi trường (env/dev, env/prod...), đặt file terraform.tfvars để tập trung giá trị riêng cho từng môi trường: region, tên tài nguyên, v.v.​
- Khi muốn apply cho một môi trường, dùng lệnh `terraform apply -var-file=env/dev/terraform.tfvars` hoặc chuyển vào thư mục `env/dev` và dùng `terraform -chdir=../../ apply -var-file=env/dev/terraform.tfvars`

### 4.1 Scnerio 2: Các môi trường có sự khác nhau về resources

Cách tổ chức thư mục

```
Terraform-project/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── Terraform.tfvars
│   │   ├── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── Terraform.tfvars
│       ├── backend.tf
├── templates/
│   └── config.tftpl
├── providers.tf
├── versions.tf
└── README.md
```

- Chia nhỏ infrastructure thành các module reusable (vpc, compute, database). Mỗi module có 1 trách nhiệm duy nhất và module không nên gọi module khác trực tiếp
- Cô Lập Môi Trường: Mỗi environment (dev/staging/prod) có backend riêng. Sử dụng Terraform.tfvars để override variables
- File main.tf của từng module sẽ refer đến file variables.tf tương ứng trong cùng thư mục. Có thể override variable của module trực tiếp từ file main.tf của environment (hoặc trong main.tf của environment refer đến variables.tf của environment, rồi variables.tf lại refer đến terraform.tfvars)

## 5. Terraform state

Workflow hoạt động của Terraform 

- Khi chạy `Terraform init` -> download tất cả các provider plugin được khai báo
- Chạy `Terraform plan` lần đầu tiên, Terraform sẽ nhận thấy không có state record -> hiểu rằng chưa có resource và Terraform chỉ cần tạo mới resource
- Chạy `Terraform apply` -> refresh in-memory state 1 lần nữa và nhận ra chưa có file state record, cần tạo mới -> nhấn yes để tạo mới
- Nếu run `Terraform apply` 1 lần nữa mà configuration file không bị thay đổi thì Terraform sẽ không có action gì . Terraform nhận biết được trạng thái thực tế hiện tại của cơ sở hạ tầng do Terraform quản lý thông qua file Terraform.tfstate (được tạo ra khi chạy Terraform apply lần đầu), file này lưu giữ trạng thái của resource từ lệnh `Terraform apply` trước. Terraform xem state file như single source of truth để nhận biết trạng thái của resources mà không cần phải truy vấn lên hạ tầng thật (gây mất thời gian). Best practice là ta nên lưu state file ở remote machine để cả team dùng chung
- Nếu ta thay đổi configuration file và chạy `Terraform apply` 1 lần nữa thì Terraform sẽ nhận biết được desired state khác với real-world state -> thực hiện modify resource cho phù hợp
- Nếu sửa hạ tầng một cách thủ công, tức là thay đổi trực tiếp trên tài nguyên bên ngoài ngoài Terraform quản lý, sẽ gây ra sự khác biệt giữa trạng thái thực tế của hạ tầng và trạng thái lưu trong file Terraform.tfstate do Terraform quản lý. Khi đó, khi chạy lại lệnh Terraform plan hoặc Terraform apply, Terraform sẽ phát hiện ra sự không đồng bộ này và sẽ hiển thị các thay đổi hoặc cố gắng sửa lại tài nguyên để đưa về trạng thái đúng theo mã cấu hình Terraform.


## 7. Datasource
- Data sources giúp lấy thông tin các tài nguyên hiện có trên môi trường như VPC, subnet, security group trong tài khoản AWS . Đây là cách phổ biến để tham khảo các tài nguyên đã tồn tại và dùng trong cấu hình.
- Data source chỉ đọc thông tin, không thể quản lý các tài nguyên đấy (create,update,delete)
- Ví dụ

**Lấy default VPC trong vùng hiện tại**
```
data "aws_vpc" "default" {
  default = true
}
```

**Lấy danh sách subnet trong VPC đó**
```
data "aws_subnets" "default_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
```

**Lấy security group mặc định trong VPC**
```
data "aws_security_group" "default_sg" {
  vpc_id = data.aws_vpc.default.id
  name   = "default"
}
```

**Ví dụ lấy tất cả subnet trong một VPC nhất định**

```
data "aws_subnets" "selected" {
  filter {
    name   = "vpc-id"
    values = ["vpc-12345678"]  # Thay bằng VPC ID của bạn
  }
}
```

**Sử dụng danh sách subnet trong các resource**
```
resource "aws_autoscaling_group" "example" {
  name                      = "example-asg"
  launch_configuration      = aws_launch_configuration.example.name
  vpc_zone_identifier       = data.aws_subnets.selected.ids
  min_size                  = 1
  max_size                  = 3
  desired_capacity          = 2
}
```





## 8. Meta arguments

Meta-argument trong Terraform là một loại đối số đặc biệt được tích hợp sẵn trong ngôn ngữ cấu hình Terraform, nhằm điều khiển cách Terraform tạo và quản lý cơ sở hạ tầng. Các meta-argument có thể được dùng trong mọi loại tài nguyên (resource) và cả trong các khối module, cho phép kiểm soát vòng đời của tài nguyên như hành vi khi tạo mới, cập nhật, hay phá hủy tài nguyên, cũng như thiết lập dependency giữa các tài nguyên.

Các meta-argument phổ biến trong Terraform bao gồm: count, for_each, depends_on, provider, lifecycle, loop

### 8.1. Count
- Sử dụng để tạo nhiều bản sao của cùng một tài nguyên/module dựa trên số lượng nguyên (integer) chỉ định.
- `Count` giúp giảm việc viết mã lặp lại và quản lý nhiều instance tài nguyên một cách linh hoạt.

Ví dụ 1:
```
resource "aws_instance" "example" {
  ami           = "ami-0078ef784b6fa1ba4"
  instance_type = "t2.micro"
  count         = 3 #tạo ra 3 EC2 instance
}
```

Ví dụ 2:
```
variable "sandboxes" {
  type    = list(string)
  default = ["sandbox_server_one", "sandbox_server_two", "sandbox_server_three"]
}

resource "aws_instance" "sandbox" {
  ami           = "ami-0078ef784b6fa1ba4"
  instance_type = "t2.micro"
  count         = length(var.sandboxes) #giá trị của count phụ thuộc vào số phần tử trong chuỗi "sandboxes", giúp dễ dàng thay đổi code

  tags = {
    Name = var.sandboxes[count.index] #count.index là một biến đặc biệt có sẵn do Terraform tự động tạo ra bên trong resource khi sử dụng thuộc tính count. Nó đại diện cho chỉ số (index) của bản sao resource hiện tại mà Terraform đang xử lý, bắt đầu từ 0, tăng lên 1 theo từng bản sao.
  }
}
```

**Nhược điểm khi sử dụng Count**

count﻿ phải được biết trước khi Terraform thực hiện bất kỳ thao tác tạo hoặc thay đổi tài nguyên nào. Điều này có nghĩa là count﻿ không thể dựa vào các giá trị chỉ có được sau khi tài nguyên được tạo, như ID duy nhất của tài nguyên trên hệ thống bên ngoài. Đây là hạn chế lớn khi cần tạo tài nguyên dựa trên các giá trị động hoặc runtime.​

Nếu một phần tử ở giữa danh sách mà count﻿ dùng để tạo các instance bị xóa, các instance phía sau sẽ bị dịch chuyển chỉ số (index) xuống, dẫn đến việc Terraform phá hủy và tạo lại các tài nguyên đó không cần thiết. Hiện tượng này gọi là "index shifting problem", làm tăng rủi ro gián đoạn và không ổn định cho hạ tầng.​

count﻿ chủ yếu hỗ trợ tạo nhiều instance giống nhau, nên không phù hợp khi cần quản lý các tài nguyên khác biệt nhau về cấu hình, cho trường hợp đó for_each﻿ sẽ linh hoạt và phù hợp hơn.

Việc hardcode giá trị count﻿ trong mã làm giảm tính linh hoạt và khó bảo trì, vì khi muốn thay đổi số lượng instance phải sửa mã nguồn và dễ xảy ra lỗi nếu không thống nhất quản lý giá trị này.​

Một số expressions phức tạp hoặc giá trị nhạy cảm không thể được dùng trực tiếp trong meta-argument count﻿.

Tuy nhiên count có 1 downside khi update là khi ta xóa resource ở index 1 thì index của các resource khác sẽ thay đổi -> tất cả đều phải recreate (hỏi thêm chatgpt để viết lại). Dùng for each có thể giải quyết vấn đề này
Ta có thể output ra để xem resource "pet" sẽ là 1 list
### 8.2. For each

for_each: Tương tự count nhưng cho phép tạo nhiều instance dựa trên một map hoặc set với khóa duy nhất, giúp quản lý từng instance riêng biệt.

```
resource "local_file" "pet" {
  filename = each.value
  for_each = toset(var.filename) #do for_each chỉ làm việc với map hoặc set nên cần phải convert sang dạng set 
}
```
Ta có thể output ra để xem resource "pet" sẽ là 1 map


depends_on: Xác định rõ các phụ thuộc giữa các tài nguyên, buộc Terraform phải tạo tài nguyên này sau khi tài nguyên khác đã được tạo xong.

provider: Chỉ định nhà cung cấp dịch vụ (provider) cụ thể áp dụng cho tài nguyên/module, hữu ích khi dùng nhiều provider hoặc nhiều cấu hình provider.

### 8.3. Lifecycle rule trong Terraform

lifecycle: Khối meta-argument chứa các thiết lập kiểm soát vòng đời tài nguyên như create_before_destroy (tạo tài nguyên mới trước khi phá tài nguyên cũ), prevent_destroy (ngăn phá hủy tài nguyên quan trọng), và các tùy chọn bỏ qua thay đổi thuộc tính (ignore_changes).

- Mặc định khi update 1 resource, Terraform sẽ xóa resource đấy trước sau đó mới recreate lại
- Để thay đổi default behavior đấy thì dùng life cycle block
- Các lifecycle có thể dùng:
  - create_before_destroy: tạo resource trước rồi mới xóa resource cũ
  - prevent_destroy: không xóa resource cũ. Nếu resource đấy bắt buộc phải xóa mới update được thì lệnh Terraform apply sẽ bị lỗi. VD áp dụng với database
  - ignore_changes: chỉ định Terraform "bỏ qua" một hoặc một số attributes nhất định khi kiểm tra thay đổi của resource đó trong quá trình apply. Nói cách khác, nếu thuộc tính được khai báo trong ignore_changes bị thay đổi ngoài Terraform hoặc do điều kiện bên ngoài, Terraform sẽ không cố gắng cập nhật hay tạo lại resource chỉ vì thay đổi của những thuộc tính này. Có thể nhận vào 1 list attribute hoặc `ignore_changes = all`. VD `ignore_changes = [tags,ami]` thì khi ta thay đổi tag của EC2 manually, lệnh `Terraform apply` sẽ không sửa lại tag của EC2 đấy cho đúng với state file. Use case thường là bỏ qua thay đổi các tag trên resource hoặc các thuộc tính được tự động cập nhật, giúp Terraform không liên tục phát hiện và thực hiện các thay đổi không cần thiết

- Ví dụ:
```
resource {
  ...
  lifecycle {
    create_before_destroy = true #tạo resource trước rồi mới xóa resource cũ
  }
}
```

## 9. Version constraint

Dùng để chỉ định version của provider thay vì dùng version latest
```
Terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "1.4.0" #hoặc có thể dùng != , > , < hoặc kết hợp VD: > 1.2.0, < 2.0.0, != 1.4.0
    }
  }
}
```


## 10. Terraform with AWS

Best pratice để đảm bảo bảo mật khi làm việc với terraform
- Tạo IAM role dành riêng cho terraform với các quyền phù hợp. VD role terraform-admin với quyền AdministratorAccess để provision resource, hoặc role terraform-viewer với quyền ViewOnlyAccess để thực thi plan. Role đấy có trust entity là AWS account - "This account" (mục đích là để các user trong account của mình có thể assume terraform role)
- Tạo 1 user với quyền chỉ cho phép assume terraform role
- Thêm profile terraform-admin hoặc terraform-viewer vào trong file ~/.aws/config với nội dung như dưới và thay provider thành profile mới này

```
[profile terraform-admin] #tên profile có thể đặt tuỳ ý
role_arn = arn:aws:iam::424432388155:role/terraform-admin
source_profile = my-user
```
- Mục đích của việc khai báo như trên: Khi chạy lệnh với option `--profile terraform-admin`, AWS CLI sẽ dùng credentials từ profile my-user để assume role terraform-admin và mọi lệnh thực hiện sẽ chạy dưới quyền của IAM role terraform-admin. Lưu ý:
  - Chỉ cần khai báo credential cho profile my-user, không cần khai báo credentials trực tiếp cho profile terraform-admin trong file cấu hình AWS CLI.
  - `my-user` cần được cấp quyền thực hiện hành động sts:AssumeRole lên role terraform-admin.
  - Nếu muốn triển khai các resources trong thư mục env/dev thì cần đảo bảo trong thư mục env/dev cũng có file provider.tf Lý do là Terraform chỉ đọc cấu hình trong thư mục đang chạy, khi dùng option `-chdir=env/dev`, Terraform sẽ dùng file cấu hình và credential theo nội dung trong env/dev, không tự động lấy theo root

Ví dụ về IAM
```
provider "aws" {
  region     = "us-west-2"
  access_key = "AKIAI44QH8DHBEXAMPLE"
  secret_key = "je7MtGbClwBF/2tk/h3yCo8n..."
}

resource "aws_iam_user" "admin-user" {
  name = "lucy"
  tags = {
    Description = "Technical Team Leader"
  }
}
resource "aws_iam_policy" "adminUser" {
  name   = "AdminUsers"
  policy = <<EOF  #hoặc thay vì đưa policy trực tiếp vào đây, ta có thể lưu thông tin ra 1 file policy.json khác rồi refer đến bằng hàm file : policy = file("policy.json")
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "*",
        "Resource": "*"
      }
    ]
  }
  EOF
}

resource "aws_iam_user_policy_attachment" "lucy-admin-access" {
  user       = aws_iam_user.admin-user.name
  policy_arn = aws_iam_policy.adminUser.arn
}

```

Tuy nhiên cách trên sẽ thiếu bảo mật do đưa credential vào trong configuration file. Thay vì thế ta nên cài credential trong aws cli trên server chạy Terraform . Lưu ý vẫn cần giữ cụm provider với region. Hoặc tạo file riêng provider.tf chứa block provider


Ví dụ về S3
```
resource "aws_s3_bucket" "finance" {
  bucket = "finanace-21092020"
  tags    = {
    Description = "Finance and Payroll"
  }
}

resource "aws_s3_bucket_object" "finance-2020" {
  content = "/root/finance/finance-2020.doc"
  key     = "finance-2020.doc"
  bucket  = aws_s3_bucket.finance.id
}

data "aws_iam_group" "finance-data" {
  group_name = "finance-analysts"
} #Terraform lấy thông tin về IAM group có tên "finance-analysts" đã tồn tại trên AWS để sử dụng trong cấu hình Terraform, nhưng không tạo mới nhóm IAM này.

resource "aws_s3_bucket_policy" "finance-policy" {
  bucket = aws_s3_bucket.finance.id
  policy = <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": "*",
        "Effect": "Allow",
        "Resource": "arn:aws:s3:::${aws_s3_bucket.finance.id}/*",
        "Principal": {
          "AWS": [
            "${data.aws_iam_group.finance-data.arn}"
          ]
        }
      }
    ]
  }
EOF
}
```

## 11. Remote state file

##### Nhược điểm của việc lưu trữ state file trên máy local trong Terraform gồm các vấn đề sau:
- Dễ gây xung đột khi làm việc nhóm: Khi nhiều người cùng quản trị hạ tầng, việc lưu state file trên máy cá nhân dễ dẫn đến xung đột, overwrite hoặc mất đồng bộ trạng thái hạ tầng.
- Nguy cơ mất mát, hỏng hoặc xóa dữ liệu: Nếu máy tính gặp sự cố, mất file state local sẽ khiến quản lý hạ tầng không chính xác, khó phục hồi hay khôi phục lại trạng thái cũ.
- Không hỗ trợ tính năng locking và bảo vệ concurrent actions: Local backend không khoá state file, dễ bị lỗi hoặc phá vỡ khi có nhiều thao tác song song hoặc nhiều người cùng apply, gây ra nguy cơ corrupt hoặc state drift (lệch trạng thái thực tế).
- Rủi ro về bảo mật State file lưu trên máy cá nhân có thể chứa thông tin nhạy cảm (secret, password, endpoint...), dễ bị lộ hoặc truy cập trái phép nếu không kiểm soát tốt.
- Không có versioning & audit trail: Local backend không quản lý version hoặc lịch sử thay đổi file state, dẫn đến khó rollback khi gặp lỗi hoặc truy vết lịch sử hạ tầng


##### Nhược điểm của việc lưu trữ state file trên git:
- Không hỗ trợ tính năng locking và bảo vệ concurrent actions

Giải thích thêm về cơ chế lock state file: lock state file giúp đảm bảo an toàn và nhất quán cho state file nhiều người, hoặc nhiều tiến trình, cùng thực hiện thao tác thay đổi hạ tầng. Cách thức khóa state file: Khi một lệnh Terraform apply hoặc Terraform plan chạy, Terraform sẽ thực hiện thao tác lock trên state file, lock này giữ quyền truy cập state file cho duy nhất một tiến trình tại một thời điểm; các thao tác khác sẽ bị chặn hoặc báo lỗi "state locked" và phải đợi tiến trình đang chiếm giải phóng lock.

##### Lưu state file trên remote backend có các ưu điểm:

- Hạn chế xung đột khi làm việc nhóm: Khi dùng local backend, state chỉ lưu trên máy cá nhân, dễ gây xung đột nếu nhiều người cùng vận hành Terraform trên một project dẫn đến thay đổi hạ tầng không kiểm soát được. Remote backend có cơ chế lock file trạng thái, ngăn không cho nhiều user cùng lúc thực hiện thao tác apply, giảm nguy cơ conflict.
- Tăng tính bảo mật và kiểm soát truy cập: State file có thể chứa sensitive data như password, endpoint, thông tin bảo mật, do đó việc lưu trên remote backend (ví dụ như S3, Terraform Cloud...) với phân quyền phù hợp sẽ bảo vệ dữ liệu tốt hơn so với lưu trên máy local.
- Quản lý tập trung, dễ backup và phục hồi: Khi lưu trạng thái hạ tầng trên remote backend, toàn bộ team đều tham chiếu cùng một nguồn lưu trữ tập trung, giúp đồng bộ hóa trạng thái và dễ dàng thực hiện backup, restore khi cần thiết
- Hỗ trợ CI/CD và tự động hóa: Việc lưu state file trên remote backend giúp tích hợp dễ dàng vào các pipeline CI/CD, không phụ thuộc vào thiết bị cá nhân của từng thành viên, giảm rủi ro mất dữ liệu khi thay đổi máy làm việc hoặc khi cần chạy tự động hóa.

##### Cách khai báo remote state file
```
Terraform {
  backend "s3" {
    bucket         = "kodekloud-Terraform-state-bucket01"
    key            = "finance/Terraform.tfstate"
    region         = "us-west-1"
    dynamodb_table = "state-locking" #optinal, DynamoDB table được dùng làm cơ sở dữ liệu để lưu trạng thái khóa (lock), đảm bảo chỉ có một tiến trình duy nhất được phép chỉnh sửa state file tại một thời điểm. Khi một thao tác apply hoặc plan bắt đầu, Terraform sẽ tạo một khóa trong DynamoDB và giữ quyền truy cập; các thao tác khác phải chờ cho đến khi khóa được giải phóng.
  }
}
```

##### Các lệnh làm việc với state

Syntax: Terraform state <subcommand> [options] [args]
Trong đó subcommand có thể là list, mv, pull, rm, show
VD:
- Terraform state list -> show ra tất cả resources trong state file (không show chi tiết)
- Terraform state show aws_s3_bucket.mybucket -> show ra chi tiết resource mybucket
- Terraform state mv -> dùng để đổi tên resource trong state file (lưu ý cần thay đổi manually trong configuration file) hoặc move item từ state file này sang state file khác
- Terraform state pull -> download và show ra remote state file
- Terraform state rm <resource> -> dùng để xóa item ra khỏi state file (lưu ý resource vẫn tồn tại trên môi trường thật)

### Provisioner
Provisioner trong Terraform là một tính năng cho phép thực thi các đoạn script hoặc lệnh sau khi tài nguyên (resource) được Terraform tạo ra. Provisioner có thể chạy script ở máy local (máy đang chạy Terraform) hoặc trên máy remote (ví dụ như máy chủ EC2 vừa mới tạo). Provisioner thường được dùng để cấu hình hạ tầng sau khi nó đã được tạo, ví dụ như cài đặt phần mềm, chỉnh sửa tập tin cấu hình, hoặc chạy các lệnh khởi tạo.

Có hai loại provisioner chính trong Terraform:

1. local-exec: Chạy các lệnh trên máy local, nơi Terraform đang chạy.
```
resource "aws_instance" "webserver" {
  ami           = "ami-0edab43b6fa892279"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo ${aws_instance.webserver.public_ip} >> /tmp/ips.txt"
  }
}
```

2. remote-exec: Chạy các lệnh trên máy remote, như máy chủ mới được provision.

```
resource "aws_instance" "webserver" {
  ami           = "ami-0edab43b6fa892279"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install nginx -y",
      "sudo systemctl enable nginx",
      "sudo systemctl start nginx",
    ]
  }
  connection {
    type = "ssh"
    host = self.public_ip
    user = "ubuntu"
    private_key = file("/root/.ssh/web")
  }
  key_name               = aws_key_pair.web.id
  vpc_security_group_ids = [ aws_security_group.ssh-access.id ]
}
```

Provisioner thường được thực thi khi tài nguyên được tạo ra (creation-time provisioners) hoặc khi tài nguyên bị xóa (destruction-time provisioners - dùng option `when = destroy`). Ngoài ra, nếu provisioner chạy thất bại, Terraform có thể đánh dấu tài nguyên đó là không thành công, hoặc có thể được cấu hình để tiếp tục. (dùng chỉ `on_failure` , mặc định là fail, có thể set thành continue) 

Provisioner giúp đảm bảo rằng quá trình tạo hạ tầng không chỉ dừng lại ở việc tạo tài nguyên mà còn thực hiện các bước cấu hình cần thiết, làm cho việc tự động hóa hạ tầng trở nên hoàn chỉnh hơn và kiểm soát chặt chẽ hơn về trạng thái thành công của việc provision.

Ví dụ, khi tạo một máy ảo EC2, provisioner remote-exec có thể dùng để cài đặt Apache HTTP Server ngay sau khi máy được tạo ra, và Terraform sẽ chỉ đánh dấu tài nguyên đó là thành công khi lệnh cài đặt thành công


## 12. Module
- Terraform module là một nhóm các file cấu hình Terraform được đóng gói lại thành một đơn vị riêng biệt, có thể tái sử dụng. Mỗi module đại diện cho một phần hoặc thành phần cụ thể của hạ tầng (ví dụ: một VPC, một nhóm compute instance, một database) và giúp tổ chức, quản lý cấu hình một cách hiệu quả, tránh lặp lại mã nguồn.
- Terraform modules là cách hiệu quả để tái sử dụng mã và quản lý hạ tầng một cách có tổ chức trong các dự án lớn bằng cách cho phép viết cấu hình một lần và dùng lại nhiều lần trong các dự án hoặc môi trường khác nhau, giúp tiết kiệm thời gian và giảm sai sót.
- Module có thể chia sẻ giữa các nhóm, các dự án hoặc tải module từ cộng đồng

### 12.1. Cấu trúc thư mục module
```
Terraform-project/
├── modules/
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
├── env/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── Terraform.tfvars
│   │   ├── backend.tf
├── providers.tf
├── versions.tf
└── README.md
```

**Nội dung file modules/compute/main.tf**
```
data "aws_ami" "amazon_linux" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["amazon"]
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_instance" "mytestinstance" {
    ami = data.aws_ami.amazon_linux.id
    instance_type = var.instance_type
    availability_zone = data.aws_availability_zones.available.names[var.index] #lấy az name theo index. Biến index được khai báo trong file modules/compute/variables.tf
}
```

**Nội dung file modules/compute/variables.tf**
```
variable "instance_type" {
}
variable "index" {
    default = 0
}
```

**Nội dung file env/dev/main.tf**
```
module "vpc" {
    source = "../../modules/vpc"
module "compute" {
    source = "git@github.com:truongton812/lab-terraform.git?ref=0.1.2" #bên cạnh chỉ định folder còn có thể chỉ định git repo kèm tag version, hữu ích trong trường hợp nhiều môi trường cùng refer đến 1 module và ta không muốn làm ảnh hưởng đến toàn bộ các môi trường khi thay đổi module 
    instance_type = var.instance_type #hoặc có thể khai báo trực tiếp instance_type ở đây. VD instance_type = "t2.nano"
    index = var.az_index #index lấy giá trị từ biến az_index trong file env/dev/variables.tf. Sau đó giá trị của index sẽ được override vào biến index ở modules/compute/variables.tf
    vpc_id = module.vpc.vpc_id #lấy output từ vpc module ở trên
}
```

**Nội dung file env/dev/variables.tf**
```
variable "instance_type" {
}

variable "az_index" {
}
```
**Nội dung file env/dev/terraform.tfvars**
```
cidr_block = "172.18.0.0/22"
instance_type = "t2.nano"
az_index = 2
```

## 13. Import

- Là tính năng cho phép đưa cácresources đã được tạo thủ công hoặc ngoài phạm vi quản lý của Terraform vào trong Terraform state, từ đó quản lý chúng hoàn toàn bằng Terraform.​
- Sau khi import, Terraform sẽ theo dõi trạng thái tài nguyên đó giống như các tài nguyên được khởi tạo hoàn toàn qua Terraform.​
- Import chủ yếu để phục vụ việc chuyển đổi quản lý hạ tầng sang Terraform mà không phải xóa rồi tạo lại từ đầu.​
- Quy trình sử dụng
  - Khai báo resource trong file .tf tương ứng với tài nguyên cần import do terraform import không tự động sinh ra code cấu hình .tf cho resource đã import
  - Chạy lệnh terraform import <resource_type>.<resource_name> <real_resource_id>, ví dụ: `terraform import aws_instance.example i-1234567890abcdef0`. Hoặc có thể khai báo import block trong file .tf
- Khi không muốn dùng terraform để quản lý resource nữa thì sử dụng lệnh `terraform state rm <resource_type>.<resource_name>` -> resource sẽ bị xóa ra khỏi file state của Terraform. Lệnh này áp dụng cho cả resource được import và resource được tạo bởi Terraform. Lưu ý resource ngoài thực tế (ví dụ máy EC2 thực tế) vẫn còn nguyên, không bị xóa trên AWS hoặc môi trường cloud của bạn.​

### Dùng Terraform để deploy helm chart

Sử dụng provider là Helm

Example: ví dụ cấu hình Terraform sử dụng provider Helm để cài đặt ứng dụng Grafana lên Kubernetes cluster local thông qua Helm chart

```
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "grafana" {
  name       = "grafana"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana"
  version    = "7.0.6"

  set { #Gán giá trị cho biến của chart (ở đây cấu hình service thành loại NodePort)
    name  = "service.type" 
    value = "NodePort"
  }
}
```

Example: ví dụ cấu hình Terraform sử dụng provider Helm để cài đặt ứng dụng Grafana lên Kubernetes cluster trên AWS thông qua Helm chart

```
data "aws_eks_cluster" "eks" {
  name = aws_eks_cluster.eks.name #hoặc có thể dùng trực tiếp tên của cluster trên AWS
}
data "aws_eks_cluster_auth" "eks" {
  name = aws_eks_cluster.eks.name #hoặc có thể dùng trực tiếp tên của cluster trên AWS
}

#Dùng data source sẽ giúp helm provider chờ cho đến khi cluster được tạo xong (trong TH tạo cluster cũng bằng Terraform)

provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.eks.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.eks.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.eks.token #dùng token để kết nối đến k8s cluster
  }
}
```
