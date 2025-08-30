

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
