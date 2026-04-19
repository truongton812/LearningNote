# Terragrunt

Cấu trúc
```
project/
├── terragrunt.hcl                    ← config chung, viết 1 lần
│
├── environments/
│   ├── configuration/
│   │   ├── dev/
│   │   │   ├── us-east-1/
│   │   │   │   └── network/
│   │   │   │       └── terraform.tfvars
│   │   │   ├── us-west-1/
│   │   │   │   └── network/
│   │   │   │       └── terraform.tfvars
│   │   ├── prod/
│   │   │   ├── us-east-1/
│   │   │   │   └── network/
│   │   │   ├── us-west-1/
│   ├── dev/
│   │   ├── us-east-1/
│   │   │   └── terragrunt.hcl
│   │   └── us-west-1/
│   │       ├── network/
│   │       │   └── terragrunt.hcl
│   │       └── app/
│   │           └── terragrunt.hcl
│   ├── prod/
│   │   ├── env.hcl                   ← env = "prod"
│   │   ├── us-east-1/
│   │   │   ├── network/
│   │   │   │   └── terragrunt.hcl
│   │   │   └── app/
│   │   │       └── terragrunt.hcl
│   │   └── us-west-1/
│   │       ├── network/
│   │       │   └── terragrunt.hcl
│   │       └── app/
│   │           └── terragrunt.hcl
│   └── common.hcl
│
└── modules/                          ← terraform thuần, tái sử dụng
    ├── network/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── output.tf
    └── app/
        ├── main.tf
        ├── variables.tf
        └── output.tf
```

File terragrunt.hcl trong mỗi child dir
```
include "root" { #include dùng để inheritate file terragrunt.hcl
  path = find_in_parrent_folder("common.hcl") #nếu ko khai báo tên thì mặc định tìm file terragrunt.hcl
}

terraform { #config how terragrunt interacts with terraform
  source = "../../..//modules/network" #load module
  extra_arguments "custom_vars" { #đặt tên gì cũng đc
    # Apply these extra arguments to these Terraform commands #extra argument chỉ được apply vào các lệnh terraform dưới đây
    commands = [
      "plan",
      "apply",
      "destroy",
      "import",
      "push",
      "refresh"
    ]
    # Bắt buộc phải tồn tại file variable này, nếu ko sẽ lỗi. File này được load vào module
    required_var_files = ["${get_parent_terragrunt_dir()}/configuration/${basename(dirname(dirname(get_terragrunt_dir())))}/${basename(dirname(get_terragrunt_dir()))}/${basename(get_terragrunt_dir())}/network.tfvars"]
    # Automatically load these tfvars files if they exist
    optional_var_files = [
      "${get_terragrunt_dir()}/../common.tfvars",
      "${get_terragrunt_dir()}/terraform.tfvars",
    ]
  }
```


File common.hcl ở root
```
remote_state { #block để set remote state 
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```


## Các built-in fuction của terraform (đưa vào bài terraform thì đúng hơn)
- basename("path/to/file") -> return last portion of path. VD basename("env/dev/us-east-1") sẽ trả về us-east-1
- dirname("path") -> remove last portion. VD dir("env/dev/us-east-1") sẽ trả về "env/dev"
- mix 2 fc trên lại sẽ lấy đc phần giữa. VD basename(dirname("env/dev/us-east-1")) trả về dev

## Built-in fc của teragrunt
- get_parent_terragrunt_dir() -> trả về đường dẫn tuyệt đối chứa file terragrunt cha (dựa theo include). Ở VD trên sẽ trả về C:/project/terragrunt/environment (do trong thư mục environment có file common.hcl)
- get_terragrunt_dir() -> trả về đường dẫn tuyệt đối của thư mục chứa file terragrunt.hcl hiện tại
