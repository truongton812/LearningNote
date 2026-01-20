https://docs.google.com/document/d/103CadDRznUzqfriz7jC2cOY4Lfx6EzEQi-Xyw_n8tJw/edit?tab=t.0

Boto3 là SDK (Software Development Kit) chính thức của AWS dành cho Python, giúp tương tác với các dịch vụ AWS như EC2, S3, DynamoDB, Lambda, v.v. thông qua code Python.

Boto3 cũng là thư viện nền phía sau aws-cli — khi dùng lệnh aws s3 cp ... hay aws ec2 describe-instances, thực ra đang dùng boto3 để gọi API.​

Cách dùng cơ bản
```python
import boto3

# Cấu hình region và credentials (thường để trong AWS config/credentials)
s3_client = boto3.client('s3', region_name='ap-southeast-1') #boto3.client('s3) làm hàm để tạo a client object cho Amazon S3,
#sau đó client object được gán vào biến s3_client. Ta có thể dùng biến s3_client để gọi các S3 method, VD s3_client.list_buckets()


# Liệt kê các bucket
response = s3_client.list_buckets()
for bucket in response['Buckets']:
    print(bucket['Name'])

# Tạo bucket mới
s3_client.create_bucket(
    Bucket='my-bucket-2026',
    CreateBucketConfiguration={'LocationConstraint': 'ap-southeast-1'}
)
```


Trong boto3, có 2 cách để tương tác với AWS service là client và resource. Client là mức thấp (low-level) còn resource là mức cao (high-level) và mang tính object-oriented hơn.
​

### 1. Boto3 Client
- Client là interface gốc, ánh xạ 1:1 với các API của AWS service (Ví dụ: list_buckets → API ListBuckets của S3).
- Trả về dữ liệu dạng thô (raw) như dict, bạn phải tự xử lý response, pagination, và toàn bộ các tham số theo chuẩn AWS API.
- Dùng client khi cần truy cập mọi API của service, hoặc cần kiểm soát chi tiết request/response (ví dụ: custom headers, complex filters).
- Client hỗ trợ tất cả các Service của AWS
- Dùng client khi:
  - Service đó không có resource (hoặc resource thiếu feature cần dùng).
  - Cần gọi API cụ thể, truyền tham số phức tạp, hoặc tích hợp với các tool khác dựa trên API AWS.
- Ví dụ

```python
import boto3

# Tạo client cho S3
s3_client = boto3.client('s3', region_name='us-east-1')

# Gọi API trực tiếp, trả về dict
response = s3_client.list_objects_v2(Bucket='my-bucket')
for obj in response.get('Contents', []):
    print(obj['Key'], obj['Size'])
```

### 2. Boto3 Resource
- Resource là lớp trừu tượng cao hơn, thiết kế theo kiểu Pythonic, object-oriented (Ví dụ: s3.Bucket(...), bucket.objects.all()).
- Cung cấp các thuộc tính (attribute) và hành động (action) như bucket.name, obj.key, obj.delete(), thay vì thao tác với dict thô.
- Tự động xử lý pagination cho các collection (ví dụ: objects.all()), và trả về dữ liệu đã được “marshall” thành kiểu Python thông thường.
- Resource không hỗ trợ tất cả các Service của AWS. Ưu tiên dùng Resource nếu Service hỗ trợ để cho code ngắn, dễ hiểu, ít lỗi do pagination hay xử lý dict.
- Ví dụ:
```python
import boto3

# Tạo resource cho S3
s3 = boto3.resource('s3', region_name='us-east-1')

bucket = s3.Bucket('my-bucket')
for obj in bucket.objects.all():  # Tự động paginate
    print(obj.key, obj.size)      # Truy cập thuộc tính như object
```
​

Lưu ý: Hoàn toàn có thể dùng cả 2 cùng lúc trong một script (ví dụ: dùng resource để thao tác với Bucket, dùng client để upload multipart với chi tiết hơn).



