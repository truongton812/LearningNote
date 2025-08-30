

### Types of IAC Tools

1. Configuration Management
Ansible

Puppet

SaltStack

2. Server Templating
Docker

HashiCorp Packer

HashiCorp Vagrant

3. Provisioning Tools
HashiCorp Terraform

CloudFormation

### terraform phase

Terraform hoạt động qua 3 phases:
- init: initilize project và identify provider. Khi chạy lệnh init thì terraform sẽ download và cài đặt plugin cho provider trong file .tf. Plugin sẽ được download về trong file .terraform/plugin (nằm trong cùng thư mục chứa file .tf)
- plan: draft a plan to get to the target state
- apply: make change to the real environment / hoặc bring environment to the desired state in case the environment is shifted from desired state
- destroy: dùng để xóa resources


### HCL (Hashicorp Configuration Language)

Syntax:
```
<block> <parameters> {
  key1 = value1
  key2 = value2
}
```

Trong đó:
- block chứa thông tin về infrastruture platform và các resources ta muốn tạo

VD để tạo file trong localhost
```
resource "local_file" "pet" {
  filename = "/root/pet.txt"
  content = "we love pets" }
```


Giải thích của chatgpt

Trong file local.tf, một resource được khai báo như sau:

```
resource "local_file" "pet" {
  filename = "/root/pets.txt"
  content  = "We love pets!"
}
```

Block Name: resource

Resource Type: local_file (trong đó “local” là provider, “file” là loại resource)

Resource Name: pet

Arguments:

filename = "/root/pets.txt"

content = "We love pets!"

### Cách tổ chức file

Trong thư mục thường đặt 1 file main.tf . Trong file đấy chứa tất cả resource cần tạo
Ngoài main.tf trong thư mục có thể đặt thêm các file:

| File Name    | Purpose                                                |
|--------------|--------------------------------------------------------|
| main.tf      | Main configuration file containing resource definition |
| variables.tf | Contains variable declarations                         |
| outputs.tf   | Contains outputs from resources                        |
| provider.tf  | Contains Provider definition                           |


### Multiple providers
trong file main.tf ta có thể sử dụng nhiều provider, VD cả aws, azure,...
-> khi chạy terraform init thì sẽ download tất cả các provider plugin được khai báo trong file main.tf (check trong .terraform/....)


### Variable trong terraform

Để dùng variable, ta khai báo trong file variable.tf
```
variable "filename" {
  default = "/root/pets.txt" #optional
  type = string #optional. Ngoài ra có thể nhận giá trị number, bool, list, map, object, tuple, set. Nếu không define thì default sẽ là any (tức có thể là bất kỳ type nào)
  description = "the file name" #optional
}
variable "content" {
  default = "We love pets!"
}
variable "prefix" {
  default = "Mrs"
}
variable "separator" {
  default = "."
}
variable "length" {
  default = "1"
}
variable "password_change" {
  default = true
  type = bool
}
```

Cách gọi variable trong main.tf

```
resource "local_file" "pet" {
  filename = var.filename
  content  = var.content
}

resource "random_pet" "my-pet" {
  prefix    = var.prefix
  separator = var.separator
  length    = var.length
}
```

##### giải thích thêm về các variable type
- list. VD:
```
variable "prefix" {
  type = list #ngoài ra có thể tạo constraint như type = list(number) -> tất cả các phần tử của list phải là number, nếu không terraform sẽ báo lỗi
  default = ["Mr", "Mrs", "Sir]
}
```
Gọi var trong file main.tf
```
resource "random_pet" "mypet" {
  prefix = var.prefix[0]
```

- map. VD:
```
variable "file-content" {
  type = map #ngoài ra có thể tạo constraint như type = map(string) -> tất cả các phần tử value của map phải là string, nếu không terraform sẽ báo lỗi
  default = {
    "statement1" = "We love pets"
    "statement2" = "We love animals"
}
```
Gọi var trong file main.tf
```
resource local_file my-pet {
  content = var.file-content["statement2"]
```

- Set: giống list nhưng các phần tử của set phải là unique, nếu không sẽ báo lỗi
VD:
```
variable "prefix" {
  type = set #ngoài ra có thể tạo constraint như type = set(string) -> tất cả các phần tử của set phải là string, nếu không terraform sẽ báo lỗi
  default = ["Mr", "Mrs", "Sir]
}
```

- Object: giống map nhưng các value của object có thể là type string/number/list/bool/... (không biết còn có thể có type khác không, nghiên cứu thêm). VD:
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

Chatgpt:

Trong Terraform, sự khác biệt giữa kiểu dữ liệu object và map như sau:

Một map là một cấu trúc dữ liệu key-value, trong đó tất cả các giá trị có cùng kiểu dữ liệu. Ví dụ map(string) có nghĩa là một bản đồ với các giá trị đều là chuỗi. Map linh hoạt, được dùng để lưu trữ dữ liệu kiểu key-value mà kiểu của tất cả giá trị phải giống nhau.

Một object định nghĩa một schema hoặc cấu trúc rõ ràng với các thuộc tính được đặt tên (named attributes), mỗi thuộc tính có thể có kiểu dữ liệu khác nhau. Ví dụ một object có thể có thuộc tính name kiểu string, age kiểu number, favorite_pet kiểu bool. Object dùng để định nghĩa kiểu dữ liệu phức tạp với nhiều trường riêng biệt, mỗi trường có kiểu riêng.

Có thể hiểu đơn giản là map là một tập hợp các giá trị cùng kiểu với key tùy ý, còn object là một kiểu dữ liệu có cấu trúc cố định với các trường đã xác định sẵn kiểu của từng trường.

Trong Terraform, một map có thể được chuyển đổi thành object nếu nó có ít nhất các key phù hợp với schema của object, nhưng có thể mất các thuộc tính dư thừa. Việc chuyển đổi ngược lại không tự động.

thuộc tính của object có thể thuộc các kiểu: string, number, bool, list, set, map, object, tuple.

- Tuple: Tuple là một tập hợp có số lượng phần tử cố định, thứ tự cố định và có thể chứa các phần tử với kiểu dữ liệu khác nhau. Tuple là bất biến (immutable), nghĩa là không thể thay đổi, thêm hoặc xóa phần tử sau khi đã tạo.

Tuple khác với list là List là một tập hợp phần tử có thứ tự, có thể thay đổi kích thước, thường chứa các phần tử cùng kiểu dữ liệu và có thể thêm, sửa, hoặc xóa phần tử trong danh sách (mutable).

VD:
```
variable kitty {
  type = tuple #ngoài ra có thể tạo constraint như type = tuple([string, number, bool])
  default = ["cat",7,true]
