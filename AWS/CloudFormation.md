# CloudFormation

## Định nghĩa
- Là dịch vụ giúp tạo và quản lý infra bằng code (IaC).
- Các khái niệm chính:
  - Template: là code để hướng dẫn CloudFormation tạo infra. Muốn update hạ tầng thì ta upload new version của template (không edit template được).
  - Stack: là 1 instance của template, chứa tất cả các resources được define trong template. Khi delete stack sẽ delete tất cả resources của stack đó.
  - Change set: là list những thay đổi khi update template ⭢ tạo change set để review trước khi thực thi.
- Work flow hoạt động:

```
         upload        refer                create        create
Template ──────▶ S3 ◀────── CloudFormation ──────▶ Stack ──────▶ resources
```
- Trong 1 template có các components:
  - AWS Template Format Version (optional)
  - Description (optional)
  - Resources (mandatory)
  - Parameters: dynamic input cho template (VD lấy từ SSM Parameter Store, Secret Manager,…)
  - Output: reference những resources đã tạo
  - Conditionals: list các điều kiện, rẽ nhánh

##  CloudFormation template’s components:

a) Resources

Là thành phần bắt buộc trong template

Resource type có dạng: service-provider::servicename::data-type-name
VD: AWS::EC2::Instance

b) Parameters

Là pre-input để đưa vào template (Lúc tạo stack sẽ trigger để user khai báo)

Parameter có các options sau:

Type: string

Number

CommaDelimitedList

List<number>

AWS-specific Parameter: để catch invalid value hoặc match với existing values trong AWS account, VD lấy key name hoặc VPC ID
