CloudWatch

Dùng để giám sát hiệu năng, cảnh báo, thu thập log, react với các events (alarm, automation) trên on-prem và cloud.

Use cases: react với state change của resource, react với metric vượt ngưỡng, react với logs system.

Các core components của CloudWatch:

CloudWatch Metric: gửi metric của resource đến CloudWatch.

CloudWatch Alarm: để react với metric vượt ngưỡng.

CloudWatch Log: quản lý log system/app.

CloudWatch EventBridge: giúp react với state change của resource.

CloudWatch Metric

Tất cả service trong AWS đều gửi metric đến CloudWatch. Metric của mỗi service sẽ có namespace (VD: EC2, Application ELB, ASG,...).

Metric có dimension để giúp phân biệt metric của object nào (VD: instance ID, environment, ...). Một metric có tối đa 30 dimensions và khi vẽ dashboard của CloudWatch sẽ cần chọn metric với dimension1, dimension2,... metric name.

Có thể tạo dashboard để nhóm các metric mong muốn (Xao Metric).

CW metrics có thể stream, gửi tới analytic, monitoring các 3rd party app như Splunk, Datadog, NewRelic theo kiểu near-real-time, có thể chọn để stream all metrics hoặc filter metrics mong muốn.

(Mặc định: EC2 gửi metric đến CW 5', 1 lần chỉ gửi được các metrics như CPUUtilization, DiskReadOps, NetworkIn, StatusCheckFailed. Nếu enable detailed EC2 monitoring: gửi metrics 1' 1 lần).

Nếu cài UCA (Unified CloudWatch Agent) thì có thể collect thêm system-level metric (memory, disk usage, ...) và log. UCA còn có thể cài trên on-prem. UCA có thể quản lý logging tập trung bằng SSM Parameter Store.

Custom Metric

Ngoài các metric có sẵn, có thể dùng PutMetricData API để tạo custom metric (bản chất UCA cũng dùng API này).


Custom metric có thể push dữ liệu thời gian ở quá khứ và hiện tại. Nếu push thời gian tương lai thì bắt buộc phải có timestamp.

Command: aws cloudwatch put-metric-data --metric-name <name> --namespace <name> --unit <unit> --value <value> --dimensions <key=value> <key=value> ... --storage-resolution <value> --timestamp <value>

Trong đó:

Storage resolution: là tần số 1 data point hiện thị trên dashboard, default là 60s. Nếu metric được push lên CW với tần số 60s thì dashboard hiển thị resolution 60s, còn có thể set parameter này.

Timestamp (optional): nếu không set thì sẽ lấy thời gian hiện tại.

Amazon Lookout for Metrics

Là AWS service dùng ML để tự động phát hiện metric bất thường, phân tích bộ chỉ số liên tục, có thể dùng phân tích metric của nhiều dịch vụ trong AWS và cả 3rd party SaaS app (thông qua app flow).

Result có thể gửi về SNS, Lambda, Slack, Webhook hoặc visualize trên AWS Console.

CloudWatch Alarm

CloudWatch Alarm – Các khái niệm

Period: là khoảng thời gian mà CloudWatch sử dụng để tổng hợp hoặc đánh giá dữ liệu của 1 metric. VD period = 5, CloudWatch sẽ xem xét dữ liệu trong khoảng 5’, để thực toán các giá trị thống kê như Average/Maximum/Minimum/Sum.

Data point: là 1 giá trị cụ thể của 1 metric tại 1 thời điểm. VD CPU của 1 EC2 tại 10h là 25% là 1 data point. 1 data point có thể bao gồm nhiều thống kê như Average/Max/Min/Sum/Percentile/SampleCount. Trong 1 period sẽ tổng hợp data point.

VD Standard monitoring gửi 1 period 5’ có 1 data point còn Detailed monitoring gửi 1 period 5’ có 5 datapoints (mỗi 1’).

Trạng thái duy nhất cho mỗi period. VD period là 5’, có 5 datapoint, CW sẽ tính giá trị trung bình của 5 điểm đó để đại diện cho period.

Thiết lập CW alarm “M out of N” nghĩa là định nghĩa số lượng data points cần vượt ngưỡng trong một N period. VD “2 out of 3 datapoints” nghĩa là cần 2 trong 3 datapoints vượt ngưỡng trong khoảng thời gian đánh giá (1 period đại diện bởi 1 datapoint).

Dùng để cảnh báo dựa trên metric.

Có 2 loại alarm: metric alarm - warning dựa trên 1 metric; composite alarm - warning dựa trên nhiều metric.

Các loại alarm state:

OK: metric trong ngưỡng

ALARM: metric quá ngưỡng / dưới ngưỡng

INSUFFICIENT_DATA: chưa đủ data

VD alarm: EC2 gửi metric đến CloudWatch -> alarm nếu metric vượt ngưỡng -> trigger ASG scale thêm.


CloudWatch Alarm có 3 target chính:
+ EC2 (stop, terminate, reboot hoặc recover)
+ Trigger ASG
+ Gửi notification đến SNS (sau đó trigger Lambda,...)

Composite alarm: monitor state của nhiều alarm kết hợp lại. Các alarm member có thể dùng OR/AND để combine vào nhau, giảm nhầm noise của composite alarm, chỉ trigger khi đồng thời CPU và network vượt ngưỡng.

EC2 instance recovery bằng CloudWatch Alarm
- Bản thân EC2 có built-in status check (health check) status check sẽ thực hiện check:
+ Instance status: check OS
+ System status: check underlying hardware
+ EBS: check EBS’s health
- Tạo thêm defined CW alarm dựa trên status check và metric, nếu alarm breached thì thực hiện recovery (thì failed metric = 1, alarm breached).

Mỗi alarm sẽ có 7 states check và Actions: create status check, EC2 recovery, assign EIP, nếu có, metadata và placement group (thay đổi ASG khi scale-out thì private và EIP thay đổi).

Có thể test alarm bằng CLI:
aws cloudwatch set-alarm-state --alarm-name <name> --state-value ALARM --state-reason <test>

CloudWatch anomaly detection
- Là dịch vụ của AWS, dùng Machine Learning để xác định normal metric pattern (shape của đồ thị), nếu metric vượt ngoài đồ thị → bất thường.
- Anomaly detection khác với đặt threshold cho metric, threshold là 1 đường thẳng ngang, còn anomaly detection là tìm pattern mang tính chu kỳ.

Logs

Để dùng CloudWatch log cần tạo log group (thường là AWS service hoặc app có 1 log group), trong log group tạo các log stream: mỗi 1 stream tương ứng với log của 1 instance/app/container.

Source gửi log vào CloudWatch log:

SDK, cloudwatch unified agent

Elastic Beanstalk: collect log của app

ECS: send container logs

Lambda: send function logs

VPC flow logs: send VPC logs

API GW: send request logs gửi đến API GW

CloudTrail: send log dựa vào filter

R53: DNS query

Từ CloudWatch log có thể gửi log tới:

S3 (batch export max 12h): dùng CreateExportTask API

Kinesis Data Stream/Firehose

Lambda

OpenSearch

Có thể áp filter vào log để đưa vào thành metric trong CloudWatch metric (dùng tab “Metric Filter”). VD: filter dòng “install failed”, mỗi lần install lỗi xuất hiện thì tăng metric; vượt ngưỡng thì alarm.

Note: metric filter là retroactive, chỉ sinh từ thời điểm filter.

CloudWatch Live Tail: tính năng để xem live log (following).


CloudWatch Log Insight

Dùng để query log trong CloudWatch log, result sẽ được visualized.

Có thể query multiple log groups across multiple accounts.

CloudWatch Logs Subscriptions

Dùng để filter log từ CloudWatch log và forward đến các dest khác. Filter được log và đẩy log near-real-time (nếu không dùng logs subscription thì chỉ batch export đi vào S3 qua API CreateExportTask).

Dùng logs subscription filter cho: ElasticSearch, KDFirehose, KDSstream, KDFirehose/KDAnalytics, EC2, Lambda.

[Diagram mô tả luồng: CW log -> logs subscription filter -> các đích như S3/OpenSearch, KDFirehose, KDSstream, Lambda]


Có thể aggregate logs từ multi-regions và multi-accounts.

Acc A: CW → Subscription filter → Subscription destination → KDSstream (acc C)
Acc B: CW → Subscription filter → acc C (nơi nhận)

Cần phải cấp quyền sub filter trong acc A/acc B send data vào destination này. IAM ở trong acc C cho phép acc A/B assume role này và action là Kinesis: put record.

EventBridge (former CloudWatch Event)

Là service giúp tuyến sự kiện (event routing) từ nhiều nguồn đến AWS service hoặc external API để có thể react với các thay đổi của AWS và non-AWS resource.

EventBridge còn dùng để tạo cronjob (VD: schedule trigger…).

Các khái niệm trong EventBridge:

Event source: là nguồn sinh ra sự kiện khi các resources trên AWS có sự thay đổi.

Event Dest: Lambda, AWS Batch, ECS task, SQS, SNS, Kinesis, Step Function, CodePipeline, CodeBuild, SSM, EC2 action (start, restart,...).

Flow: Từ event source gửi về EventBridge (filter here - optional) → EventBridge generate dạng json của event (có thể transform) → gửi đến event dest.

Event (all hoặc đc filter) trong event bus có thể archive để lưu trữ (có hạn/có thời hạn). Archived events có thể replay để debug/troubleshoot.

EventBridge có thể analyze event trong event bus và suy ra schema. Schema registry giúp generate code cho app.

Các ví dụ sử dụng EventBridge

Trigger khi có file upload vào S3, tạo Event Rule với source là S3.amazonaws.com và event type PutObject, chỉ định target là Lambda, ECS task, function.

Gửi thông báo khi EC2 bị terminated.

EventBridge cross-account

EventBridge có thể nhận và theo dõi sự kiện cross-account.

Để nhận event từ account khác thì có thể dùng default event bus hoặc tạo custom event bus, sau đó cấu hình resource policy của bus để cho phép nhận sự kiện.

Ở account nguồn thì có event tạo dùng AWS CLI/SDK gửi event sang account đích.

Để theo dõi sự kiện cross-account thì:

Trong account nguồn: tạo EventBridge, rule để theo dõi event cần và chọn target là event bus ở account đích, lưu ý cần đảm bảo IAM role của EventBridge trong account nguồn có quyền PutEvents.

Trong account đích: tạo EventBridge rule để nhận và xử lý sự kiện từ account nguồn.
