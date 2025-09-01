

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

Trong thư mục thường đặt 1 file main.tf (gọi là configuration file). Trong file đấy chứa tất cả resource cần tạo
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


##### Cách khai báo và sử dụng variable

```
variable "content" {}
```

Nếu trong file variable.tf ta không khai báo giá trị default cho variable thì ta có các cách sau để truyền giá trị cho variable: (thứ tự ưu tiên tăng dần)

1. Ta có thể truyền variable bằng cách export vào biến môi trường (luôn phải có tiền tố TF_VAR ở trước)
export TF_VAR_filename="root/pet.txt"
export TF_VAR_lenght="2"
2. Ta có thể đặt biến trong file terraform.tfvars hoặc terraform.tfvars.json (hoặc file khác chỉ cần có đuôi là *auto.tfvars hoặc *auto.tfvars.json ) trong đó chỉ chứa variable dạng `key = value`. Lưu ý nếu đặt tên khác các tên trên thì phải chỉ định file chứa variable bằng option `terraform apply -var-file <ten_file>`
3. Ta có thể truyền variable bằng option -var
VD: terraform apply -var "filename=/root/pet.txt" -var "content=We love pet"
4. Truyền khi ta chạy lệnh terraform apply sẽ có prompt để ta nhập giá trị cho variable. 

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
```

##### Output variable

Dùng để output ra thông tin VD phục vụ automation (ansible, shell script), CICD (chatgpt hỏi thêm mục đích)

```
resource "random_pet" "my-pet" {
  prefix    = var.prefix
  separator = var.separator
  length    = var.length
}

output "pet-name" {
  value       = random_pet.my-pet.id
  description = "Record the value of pet ID generated by the random_pet resource"
}
```

Khi dùng lệnh terraform output sẽ in ra tất cả output của configuration file trong thư mục hiện tại hoặc lệnh terraform output pet-name để in ra specific output



##### Refer attribute của resource

Dùng khi ta muốn lấy attribute của resource này để làm input cho resource khác. VD lấy VPC ID để đưa vào tạo EC2 instance

Lấy bằng syntax: `resource_type.resouce_name.attribute`

Hỏi chatgpt để lấy ví dụ

Khi ta định nghĩa resource reference như vậy thì mặc định ta đã implicitly chỉ định thứ tự tạo resource cho terraform

Ngoài cách implicitly còn có thể explicitly specify dependencies bằng chỉ thị   depends_on = [ resource_type.resource_name ] . Lưu ý depends_on là 1 list nên có thể chỉ định nhiều items

### Terraform state

Luồng hoạt động của terraform under the hood
- Khi chạy terraform init -> download ....
- Chạy terraform plan lần đầu tiên, terraform sẽ nhận thấy không có state record -> hiểu rằng chưa có resource và teraform chỉ cần tạo mới resource
- chạy terraform apply -> refresh in-memory state 1 lần nữa và nhận ra chưa có state record, cần tạo mới -> nhấn yes để tạo mới
- Nếu run terraform apply 1 lần nữa -> terraform sẽ không có action gì nếu configuration file không bị thay đổi . Terraform nhận biết được state hiện tại của resource thông qua file terraform.tfstate (được tạo ra khi chạy terraform apply lần đầu), file này lưu giữ trạng thái của resource từ lệnh terraform apply trước
- Nếu ta thay đổi configuration file và chạy terraform apply 1 lần nữa thì terraform sẽ nhận biết được desired state khác với real-world state -> thực hiện modify resource cho phù hợp


Chatgpt: File terraform.tfstate trong Terraform là một file trạng thái (state file) dùng để lưu trữ thông tin về trạng thái hiện tại của cơ sở hạ tầng do Terraform quản lý. Nó ghi lại cấu hình hiện tại của các tài nguyên (resources) đã được tạo hoặc thay đổi khi thực thi các lệnh như terraform apply. File này cho phép Terraform theo dõi và biết được những thay đổi cần thực hiện khi bạn chạy các lệnh tiếp theo, giúp đồng bộ trạng thái giữa file cấu hình và hạ tầng thực tế. State file có thể coi như là metadata của resource (VD xem được dependencies)
Terraform xem state file như single source of truth để nhận biết trạng thái của resources mà không cần phải truy vấn lên hạ tầng thật (gây mất thời gian). Best practice là ta nên lưu state file ở remote machine để cả team dùng chung

Nếu sửa hạ tầng một cách thủ công, tức là thay đổi trực tiếp trên tài nguyên bên ngoài ngoài Terraform quản lý, sẽ gây ra sự khác biệt giữa trạng thái thực tế của hạ tầng và trạng thái lưu trong file terraform.tfstate do Terraform quản lý. Khi đó, khi chạy lại lệnh terraform plan hoặc terraform apply, Terraform sẽ phát hiện ra sự không đồng bộ này và sẽ hiển thị các thay đổi hoặc cố gắng sửa lại tài nguyên để đưa về trạng thái đúng theo mã cấu hình Terraform.


### Các lệnh làm việc với terraform
- terraform validate -> dùng để kiểm tra syntax
- terraform fmt -> format lại terraform configuration file cho dễ đọc
- terraform show -> print current state của infrastructure . Thêm option -json để đọc dưới dạng json
- terraform providers -> xem tất cả providers có trong thư mục hiện tại
- terraform providers mirror /path/to/other/configuration/directory -> copy provider ở current directory sang directory khác
- terraform output -> in ra tất cả output trong thư mục configuration directory hiện tại. Thêm tên của output variable để chỉ lấy output của variable đấy
- terraform apply -refresh-only -> dùng để sync file state với real-world state. Dùng trong trường hợp có thay đổi manually trên real-world infrastructure thì chạy lệnh này để update state file. Lưu ý là lệnh này chỉ modify state file
- option -resfresh=false để bypass việc refresh terraform state
- terraform graph -> visualize ra mối quan hệ dependencies giữa các resource (cần phải cài phần mềm đọc format dot)

### Lifecycle rule trong Terraform

Mặc định khi update 1 resource, terraform sẽ xóa resource đấy trước sau đó mới recreate lại

Để thay đổi default behavior đấy thì dùng life cycle block

```
resource {
  ...
  lifecycle {
    create_before_destroy = true #tạo resource trước rồi mới xóa resource cũ
  }
}
```

Các lifecycle có thể dùng:
- create_before_destroy: tạo resource trước rồi mới xóa resource cũ
- prevent_destroy: không xóa resource cũ. Nếu resource đấy bắt buộc phải xóa mới update được thì lệnh terraform apply sẽ bị lỗi. VD áp dụng với database
- ignore_changes: khi có thay đổi trên real-world infra thì terraform không đưa sự thay đổi đấy về desired state theo configuration file (???). Nhận vào 1 list attribute hoặc ignore_changes = all. VD ignore_changes = [tags,ami] thì khi ta thay đổi tag của EC2 manually, lệnh terraform apply sẽ không sửa lại tag của EC2 đấy cho đúng với state file

### Datasource

Datasource giúp terraform đọc attribute từ resources nằm ngoài control của terraform. (hình như chỉ giúp đọc nội dung file, không lấy resource về để quản lý được). VD
```
data "local_file" "dog" {
  filename = "/root/dog.txt"
}
```
-> terraform sẽ tạo ra resource type là local_file từ data source (???)

| Resource                               | Data Source                       |
|--------------------------------------|---------------------------------|
| Keyword: **resource**                 | Keyword: **data**                |
| **Creates, Updates, Destroys** Infrastructure | Only **Reads** Infrastructure   |
| Also called **Managed Resources**    | Also called **Data Resources**   |

### Meta arguments

Defination:

Các meta argument: depends_on, lifecycle, count, for each, loop, 

1. Count: dùng để tạo nhiều instance của resources
```
resource "local_file" "dog" {
  filename = "/root/${var.filename[count.index]}"
  count = 3 #tuy nhiên sẽ chỉ ra 1 file do terraform tạo ra 3 file cùng 1 tên. Để fix thì cần phải tạo variable và dùng loop trên count
}
```
Cách dùng count = 3 thì sẽ hard code số lượng, nếu ta thêm file vào list filename thì sẽ vẫn chỉ có 3 files tạo ra -> nên thay bằng `count = length(var.filename)`
```
variable.tf
variable "filename" {
  default = ["pet.txt", "dog.txt", "animal.txt"]
}
```


Chatgpt: count.index là một biến đặc biệt (special variable) mà Terraform tự động tạo ra bên trong resource khi sử dụng thuộc tính count. Nó đại diện cho chỉ số (index) của bản sao resource hiện tại mà Terraform đang xử lý, bắt đầu từ 0, tăng lên 1 theo từng bản sao.

Nói cách khác, count.index giúp phân biệt từng instance/resource được tạo ra khi count được sử dụng để tạo nhiều bản sao resource cùng lúc. Đây là một biến tự động có sẵn, không phải do người dùng định nghĩa hay gọi như hàm

2. 
