

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

### Cách tổ chức file

Trong thư mục thường đặt 1 file main.tf . Trong file đấy chứa tất cả resource cần tạo
Ngoài main.tf trong thư mục có thể đặt thêm các file:
- main.tf
- variables.tf
- outputs.tf
- provider.tf

### Multiple providers
