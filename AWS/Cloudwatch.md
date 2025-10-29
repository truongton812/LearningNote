# CloudWatch

## I. Định nghĩa & Khái niệm

- CloudWatch dùng để giám sát hiệu năng, cảnh báo, thu thập log, react với các events (alarm, automation) trên on-prem và cloud.
- Use cases:
  - react với state change của resource
  - react với metric vượt ngưỡng
  - react với logs system.
- Các core components của CloudWatch:
  - CloudWatch Metric: gửi metric của resource đến CloudWatch.
  - CloudWatch Alarm: để react với metric vượt ngưỡng.
  - CloudWatch Log: quản lý log system/app.
  - CloudWatch EventBridge: giúp react với state change của resource.

## II. CloudWatch Metric
### 1. Standard metric
- Tất cả services trong AWS đều gửi metric đến CloudWatch. Metric của mỗi service thuộc 1 namespace (VD namespace EC2, ApplicationELB, ASG,...).
- Metric có dimension để giúp phân biệt metric của object nào (VD: instance ID, environment, ...). Khi tìm 1 metric trong dashboard của Cloudwatch sẽ có dạng
```
  dimension1    dimension2    ...    dimensionN    Metric Name    Alarms
   <value>        <value>              <value>       <name>
```
- Có thể tạo dashboard để nhóm các metric mong muốn (Vào Metric ➝ Action ➝ Add to dashboard).
- Cloudwath metrics có thể stream continuosly vào Kinesis Data Firehose hoặc các 3rd party app như Splunk, Datadog, NewRelic theo kiểu near-real-time. Có thể chọn để stream all metrics hoặc filter metrics mong muốn.
- Mặc định EC2 gửi metric đến Cloudwatch 5 phút 1 lần và chỉ gửi được các metrics như CPUUtilization, DiskReadOps, NetworkIn, StatusCheckFailed.
  - Nếu enable detailed EC2 monitoring có thể gửi metrics 1 phút 1 lần (mất phí)
  - Nếu cài UCA (Unified CloudWatch Agent) thì có thể collect thêm system-level metric (memory, disk usage, ...) và log. UCA còn có thể cài trên on-prem. Config của các UCA có thể quản lý tập trung bằng SSM Parameter Store.

### 2. Custom Metric
- Ngoài các metric có sẵn, có thể dùng PutMetricData API để tạo custom metric (bản chất UCA cũng dùng API này).
- Custom metric có thể push dữ liệu vào thời gian 2 tuần ngược về quá khứ và 2 giờ tới trong tương lai, không nhất thiết timestamp phải là hiện tại
- Command để push custom metric: ```aws cloudwatch put-metric-data --metric-name <name> --namespace <name> --unit <unit> --value <value> --dimensions <key=value> <key=value> ... --storage-resolution <value> --timestamp <value>```
  - Storage resolution: là tần số 1 data point hiện thị trên dashboard, default là 60s. Nếu metric được push lên Cloudwatch với tần số nhỏ hơn 60s (VD 10s push 1 lần) và ta muốn xem dashboard với resolution là 10s thì cần set parameter này.
  - Timestamp (optional): nếu không chỉ định timestamp thì sẽ lấy thời gian hiện tại.

### 3. Amazon Lookout for Metrics
- Là AWS service dùng Machine Learning để tự động phát hiện metric bất thường, phân tích và chỉ ra root cause
- Có thể dùng phân tích metric của nhiều dịch vụ trong AWS và cả 3rd party SaaS app (thông qua AppFlow).
- Result có thể gửi về SNS, Lambda, Slack, Webhook hoặc visualize trên AWS Console.

## III. CloudWatch Alarm
### 1. Các khái niệm
- Cloudwatch alarm là công cụ dùng để cảnh báo dựa trên metric
- Period: là khoảng thời gian mà CloudWatch sử dụng để tổng hợp hoặc đánh giá dữ liệu của 1 metric. VD period = 5, CloudWatch sẽ xem xét dữ liệu trong khoảng 5 phút để thực hiện tính toán các giá trị thống kê như Average/Maximum/Minimum/Sum.
- Data point: là 1 giá trị cụ thể của 1 metric tại 1 thời điểm. VD CPU của 1 EC2 tại 10h là 25% là 1 data point. 1 data point có thể bao gồm nhiều thông tin thống kê như Average/Max/Min/Sum/Percentile/SampleCount. Trong 1 period số lượng data point không cố định mà phụ thuộc vào tần số thu thập dữ liệu của metric. Ví dụ với Standard monitoring trong 1 period 5 phút có 1 data point còn với Detailed monitoring trong 1 period 5 phút có 5 datapoints (tuy nhiên khi hiển thị hoặc đánh giá trong Cloudwatch alarm, các datapoint này thường được tổng hợp thành 1 giá trị thống kê duy nhất cho mỗi period. VD nếu period là 5 phút và có 5 datapoint, Cloudwatch có thể tính giá trị trung bình của 5 điểm đó để đại diện cho period)
- Thiết lập CW alarm “M out of N” nghĩa là định nghĩa số lượng data points M cần vượt ngưỡng trong N period. VD “2 out of 3 datapoints” nghĩa là cần 2 trong 3 datapoints vượt ngưỡng trong khoảng thời gian đánh giá (1 period đại diện bởi 1 datapoint).
- Có 2 loại alarm:
  - Metric alarm - warning dựa trên 1 metric
  - Composite alarm - warning dựa trên nhiều metric.
- Các loại alarm state:
  - OK: metric trong ngưỡng
  - ALARM: metric quá ngưỡng / dưới ngưỡng
  - INSUFFICIENT_DATA: chưa đủ data
- VD alarm: EC2 gửi metric đến CloudWatch ➝ alarm nếu metric vượt ngưỡng ➝ trigger ASG scale thêm.
- CloudWatch Alarm có các target:
  - EC2 (stop, terminate, reboot hoặc recover)
  - Trigger ASG
  - Gửi notification đến SNS 
  - Trigger lambda
  - Systems Manager
- Composite alarm: monitor state của nhiều alarm kết hợp lại. Các alarm member có thể dùng OR/AND để combine vào nhau ➝ giảm "alarm noise". VD composite alarm chỉ trigger khi đồng thời CPU và network vượt ngưỡng.

### 2. Recovery EC2 instance bằng CloudWatch Alarm
- Bản thân EC2 có built-in status check (health check). Status check sẽ thực hiện check:
  - Instance status: check OS
  - System status: check underlying hardware
  - EBS: check EBS health
➝ Ta có thể define Cloudwatch alarm dựa trên status check và metric. Nếu alarm breached thì thực hiện recovery (khi failed metric = 1 ➝ alarm breached).

Cách làm: EC2 ➝ Status check ➝ Actions ➝ Create status check alarm ➝ alarm threshold = status check failed)

- EC2 recovery: sẽ tạo ra 1 EC2 có cùng private IP và public IP (cùng ENI nếu có), metadata và placement group (khác với ASG khi scale-out thì private và EIP thay đổi)
- Có thể test alarm bằng CLI:
```
aws cloudwatch set-alarm-state --alarm-name <name> --state-value ALARM --state-reason <test>
```

### 3. CloudWatch anomaly detection
- Là dịch vụ của AWS, dùng Machine Learning để xác định normal metric pattern (shape của đồ thị), nếu metric vượt ngoài đồ thị → bất thường.
- Anomaly detection khác với đặt threshold cho metric, threshold là 1 đường thẳng ngang, còn anomaly detection là tìm pattern mang tính chu kỳ.

## IV. Cloudwatch Logs
### 1. Cloudwatch logs
- Là nơi lưu trữ log của system và app (cả cloud và on-prem).
- Các loại log trong CloudWatch log:
  - Application log:
    - Là log được sinh ra từ application code, gồm custom log message, stack traces,... Log này thường được lưu dưới dạng file trên OS.
    - Nếu app running trên EC2 → cài UCA để collect và stream log vào CloudWatch log.
    - App run bằng Lambda, ECS, Fargate, Beanstalk được tích hợp để gửi log trực tiếp đến CloudWatch log.
  - OS log:
    - Là event logs, system logs mà OS generate ra (VD /var/log/messages, /var/log/auth.log).
    - Dùng UCA để stream log files vào CloudWatch log.
  - Access log:
    - Là log client access vào service (VD LB, proxy, webserver).
    - Thường mỗi service có 1 access log file riêng → dùng UCA để đẩy vào CloudWatch log.
  - AWS managed log:
    - ELB access log: log access vào LB → gửi đến S3.
    - CloudTrail logs: logs các API call → gửi đến S3 và CloudWatch logs.
    - VPC Flow logs: log traffic in/out VPC → gửi đến S3 và CloudWatch logs.
    - R53 access logs: log query R53 → gửi đến CloudWatch logs.
    - S3 access logs: log truy cập S3 → gửi đến S3 bucket khác.
    - CloudFront access logs: log request đến CloudFront → gửi đến S3.
  - Log default được encrypt, hoặc có thể encrypt bằng key của mình thông qua KMS, có thể define expiration (never expire hoặc từ 1 ngày → 10 năm).
- Để dùng CloudWatch log cần tạo log group (thường là 1 AWS service hoặc 1 application có 1 log group), trong log group tạo các log stream: mỗi 1 stream tương ứng với log của instances/files/containers.
- Source gửi log vào CloudWatch log:
  - SDK, cloudwatch unified agent
  - Elastic Beanstalk: collect log của app
  - ECS: send container logs
  - Lambda: send function logs
  - VPC flow logs: send VPC logs
  - API GW: send request logs gửi đến API GW
  - CloudTrail: send log dựa vào filter
  - R53: DNS query
- Từ CloudWatch log có thể gửi log tới:
  - S3 (batch export, max 12h): dùng CreateExportTask API
  - Kinesis Data Stream/Firehose
  - Lambda
  - OpenSearch
- Có thể áp filter vào log để đưa vào vào thành metric trong CloudWatch metric (dùng tab “Metric Filter”). VD: filter dòng “install failed” để mỗi lần “install failed” xuất hiện thì tăng metric; vượt ngưỡng thì alarm. Note: metric filter không retroactive, chỉ sinh từ thời điểm tạo filter.

### 2. CloudWatch Live Tail
- Là tính năng để xem live log (following).

### 3. CloudWatch Log Insight
- Dùng để query log trong CloudWatch log, result sẽ được visualized.
- Có thể query multiple log groups across multiple accounts.

### 4. CloudWatch Logs Subscriptions
- Dùng để filter log từ CloudWatch log và forward đến các dest khác.
- Lợi ích: Filter được log và đẩy log near-real-time (nếu không dùng logs subscription thì chỉ batch export vào S3 thông qua API CreateExportTask).
```

                                        | ---> Kinesis Data Firehose ---> S3/OpenSearch
         logs                           |
CW Log  -----> Logs Subscription Filter | ---> Kinesis Data Stream   ---> Kinesis Data Firehose/ Kinesis Data Analytics/ EC2/ Lambda
                                        |
                                        | ---> Lambda
                                        |
                                        | ---> Elasticsearch
```

- Có thể aggregate logs từ multi-regions và multi-accounts.

```
Acc A: Cloudwatch ---> Subscription filter ---> |
                                                | ---> Acc C: Subscription destination ------------> Kinesis Data Stream
Acc B: Cloudwatch ---> Subscription filter ---> |      - Cần policy cho phép subscription                   ^
                                                         filter trong acc A và acc B                        |
                                                         send data vào destination này                      | put record
                                                       - IAM trong acc C cho phép acc A/B                   |
                                                         assume role này và action là                       |
                                                         "Kinesis: put record." ----------------------------|
```

## V. EventBridge (former CloudWatch Event)
### 1. EventBridge
- Là service giúp định tuyến sự kiện (event routing) từ nhiều nguồn đến các AWS services khác hoặc external API → từ đó react với các thay đổi của AWS và non-AWS resources.
- EventBridge còn dùng để tạo cronjob (VD 1 tiếng trigger lambda 1 lần)
- Các khái niệm trong EventBridge:
  - Event: định dạng json, sinh ra khi các resources trên AWS thay đổi trạng thái.
  - Event source: sinh ra từ nhiều nguồn
    - AWS:
      - EC2, S3, ECS, Lambda, CloudTrail, CodePipeline, CodeBuild,... là các dịch vụ tự động gửi event về EventBridge mà không cần cấu hình.
      - Các dịch vụ không tự động gửi event về EventBridge, VD cloudwatch log (cần thiết lập subscription), thay đổi dữ liệu trong S3 (cần enable S3 Event Notification), thay đổi dữ liệu trong DynamoDB (cần enable DynamoDB stream).
    - Custom event của application trên AWS
    - Event của 3rd party ngoài AWS (DataDog, Zendesk, Auth0,...)
  - Event bus: nhận event và process data, trong event bus có các rule để quy định cách mà event được xử lý và định tuyến đến destination nào (SNS, Lambda, Kinesis, AWS Config,...). Có 3 loại bus:
    - Default: mặc định các AWS Services gửi event vào đây
    - Partner event bus: để nhận event từ các non-AWS resources
    - Custom: dùng khi muốn separate event
    - Note: có thể áp resource-based policy lên event bus → dùng Event bus trong 1 account làm nơi tổng hợp event của Organization.
  - Event Dest: Lambda, AWS Batch, ECS task, SQS, SNS, Kinesis, Step Function, CodePipeline, CodeBuild, SSM, EC2 action (start, restart,...).
- Flow: Từ event source gửi về EventBridge (filter here - optional) → EventBridge generate dạng json của event (có thể transform) → gửi đến event dest.
- Event (all hoặc đc filter) trong event bus có thể archive để lưu trữ (vô hạn/có thời hạn). Archived events có thể replay để debug/troubleshoot.
- EventBridge có thể analyze event trong event bus và suy ra schema. "Schema registry" giúp generate code cho app.

### 2. Các ví dụ sử dụng EventBridge
- Trigger lambda khi có file upload vào S3: tạo Event Rule với source là S3.amazonaws.com và event type PutObject, chỉ định target là Lambda
- Chạy step function khi ECS task hoàn thành
- Gửi thông báo khi EC2 bị terminated

### 3. EventBridge cross-account
- EventBridge có thể nhận và theo dõi sự kiện cross-account.
- Để nhận event từ account khác thì có thể dùng default event bus hoặc tạo custom event bus, sau đó cấu hình resource policy của bus để cho phép nhận sự kiện. Ở account nguồn, khi có event ta dùng AWS CLI/SDK để gửi event sang account đích.
- Để theo dõi sự kiện cross-account thì:
  - Trong account nguồn: tạo EventBridge rule để theo dõi event cần và chọn target là event bus ở account đích. Lưu ý cần đảm bảo IAM role của EventBridge trong account nguồn có quyền events:PutEvents.
  - Trong account đích: tạo EventBridge rule để nhận và xử lý sự kiện từ account nguồn.

---

Log group và log stream là hai khái niệm nền tảng trong AWS CloudWatch Logs, mỗi cái đóng vai trò và phạm vi quản lý khác nhau đối với dữ liệu log.

Log Group là gì?
Log group là một "ngăn chứa" (container) cho nhiều log stream.

Các log stream trong cùng một log group sẽ chia sẻ chung các thiết lập như thời gian lưu trữ (retention), quyền truy cập (access control), và các rule cho filter hoặc alarm.​

Thông thường, một log group đại diện cho một ứng dụng, một dịch vụ hoặc một thành phần logic riêng biệt trong hệ thống AWS (ví dụ: /aws/lambda/app1, /aws/ec2/webserver).​

Log Stream là gì?
Log stream là một chuỗi (sequence) các sự kiện log xuất phát từ cùng một nguồn (source).​

Mỗi nguồn cung cấp (ví dụ: mỗi instance EC2, mỗi container, mỗi Lambda execution environment...) thường sẽ tạo một log stream riêng biệt.

Bên trong log stream, mỗi sự kiện log được lưu kèm timestamp, message và các thuộc tính metadata liên quan

| Đặc điểm                  | Log Group                                         | Log Stream                                 |
|--------------------------|--------------------------------------------------|--------------------------------------------|
| Phạm vi quản lý           | Chứa nhiều log stream liên quan                   | Quản lý log events từ 1 nguồn cụ thể       |
| Cấu hình chung            | Có (retention, policies, metric filters, v.v.)    | Không (tuân theo log group chứa nó)        |
| Quan hệ                   | 1 log group : nhiều log stream                    | 1 log stream : thuộc 1 log group           |
| Ví dụ tên                 | `/aws/lambda/my-function`                         | `2025/10/29/[$LATEST]abcd1234...`          |
