Basic term
- target: là server, device,... cần monitor
- instance: là endpoint cần thu monitor. Ex: 1.2.3.4:5670, 1.2.3.4:5671
- job: là group của targets hoặc intances có điểm giống nhau
- sample: là value của 1 metric tại 1 thời điểm

# Prometheus architecture
![alt text](https://devopscube.com/content/images/2025/03/prometheus-architecture.gif)

- Prometheus server:
  - có chức năng retrieve data từ target/instance bằng cách gọi đến HTTP URL (VD http://example.com/metrics) và lưu data đấy vào storage. Prometheus hoạt động theo cơ chế pull metric định kỳ.
  - HTTP server cung cấp giao diện web quản trị, API truy vấn PromQL, đường dẫn để xem trạng thái, và đặc biệt là endpoint như /metrics để cung cấp chính các thông số hoạt động của bản thân Prometheus server. Các thành phần khác như Grafana, Prometheus Web UI hoặc API clients có thể truy vấn tới HTTP server bằng PromQL để visualize ra dashboard
- Prometheus targets: là các đối tượng (ví dụ: server, dịch vụ, ứng dụng, exporter) mà Prometheus sẽ chủ động kết nối đến để lấy (scrape) dữ liệu giám sát (metrics) theo định kỳ. Mỗi target phải cung cấp một HTTP endpoint trả về dữ liệu dạng time series, thường là đường dẫn /metrics. Nếu ứng dụng không native hỗ trợ Prometheus thì cần cài thêm exporter (ví dụ node_exporter cho server vật lý, hoặc custom exporter cho database...), hoặc cài thư viện client (ví dụ client_python, client_java...) vào ứng dụng để expose endpoint metrics cho Prometheus scrape
- Pushgateway: đa phần prometheus target không push metric, tuy nhiên có 1 số service mà Prometheus không thể dùng cơ chế pull (VD short-lived service) -> pushgateway là nơi để các service đấy đẩy metric về và lưu lại ở đấy, sau đó chờ Prometheus server pull về
- Service discovery: có 2 cách để khai báo target cần monitor cho Prometheus.
  - Khai báo target trong file cấu hình prometheus.yaml
  - Dùng service discovery. Lưu ý là nếu muốn dùng cơ chế service discovery thì Prometheus cần phải integrate với inventory database (dựa trên k8s, dns, ec2,...)
- Alertmanager: nơi nhận alert

# Cấu hình prometheus
Để cấu hình cho Prometheus thì ta khai báo trong file prometheus.yaml
```yaml
global: #define cho toàn cluster (có thể ghi đè từ target config)
  scrape_interval: 15s    # khoảng thời gian Prometheus scrape metric từ target (có thể ghi đè từ scrape_config)
  evaluation_interval: 15s # khoảng thời gian Prometheus evaluate rule define trong rules file
  scrape_timeout: 10 #khoảng thời gian scrape request bị xem là timeout

rule_files: #chỉ định location của rule file (rule là để tạo time series mới và tạo alert từ time series đấy)
  [ - <filepath_glob> ... ]

alerting: #dùng để set alert
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

scrape_configs: #khai báo info và config của target. Ta có thể ghi đè scrape_interval và các giá trị khác tại đây thay cho giá trị global
  - job_name: 'prometheus'  # Giám sát chính Prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'  # Giám sát node_exporter (giám sát máy chủ Linux)
    static_configs:
      - targets: ['localhost:9100']

```

# Query với prometheus:
- up: số lượng targets  đang đc monitor. Mặc định Prometheus monitor chính nó = cách scrape metric ở endpoint là http://localhost:9090/metrics Trả về value là 0 hoặc 1. 1 là lần last scrape thành công. Trong đó job=prometheus là do file prometheus.yaml quy định job_name=prometheus
- prometheus_http_requests_total: số lượng http request. Note: prometheus cho phép nhiều metric có cùng tên, tuy nhiên label của chúng phải khác nhau
- node_memory_MemAvailable_bytes: memory available của node


=====================
Khái niệm timeseries trong prometheus: đó là một chuỗi các điểm dữ liệu (sample) được ghi nhận liên tiếp qua các mốc thời gian, đại diện cho giá trị của một metric tại nhiều thời điểm khác nhau.
mỗi timeseries trong Prometheus được xác định duy nhất bởi:
Tên metric (ví dụ: http_requests_total)
Một tập các label (cặp key-value, ví dụ: method="POST", handler="/login")
Ví dụ định dạng một timeseries:
http_requests_total{method="POST", handler="/login"}

Trong đó:
http_requests_total là tên metric,
{method="POST", handler="/login"} là tập label

Sample là 1 điểm dữ liệu, tức là 1 giá trị tại 1 thời điểm cụ thể của metric
Ví dụ định dạng một sample:
http_requests_total{method="POST", handler="/login"} 154
-> tại thời điểm Prometheus thu thập, giá trị của http_requests_total{method="POST", handler="/login"} là 154.
Prometheus tự động thu thập (scrape) dữ liệu từ các target và lưu trữ mọi metric dưới dạng timeseries, nghĩa là mỗi giá trị đo lường luôn gắn liền với một dấu thời gian xác định.

======
Exporters
Nếu 1 application muốn đc monitor bởi prometheus, cần add instrumentation vào application code thông qua Prometheus client library. VD add instrumentation vào python app qua official python client library
Tuy nhiên nếu ta ko có quyền chỉnh sửa code hoặc cần monitor 1 server -> sử dụng exporter. Nhiệm vụ của exporter là gather required data theo yêu cầu của Prometheus, transform thành định dạng Prometheus cần và gửi lại cho Prometheus. Có nhiều loại exporter, vd node exporter cho linux machine, wm exporter cho window
- cách exporter hoạt động: exporter nhận lệnh từ prometheus để biết prometheus cần những metric gì, sau đó nó sẽ gather data từ target và chuyển thành format mà prometheus hiểu được và gửi về cho prometheus

- Node exporter: lấy metric như CPU, memory, disk space, disk I/O, network bandwidth ,... viết = go

============================
Query Prometheus bằng PromQL

- Data type in PromQL
Instance vector: a set of time series containing a single sample for each time series, all sharing the same timestamp . Chatgpt: Instance vector là một tập hợp các time series mà mỗi chuỗi chỉ chứa một giá trị dữ liệu duy nhất tại một thời điểm cụ thể
Giá trị này là giá trị gần nhất thời điểm truy vấn mà được lưu trong timeseries database. Lưu ý là truy vấn instance vector có thể truy vấn được thời gian là 1 thời điểm cụ thể, không nhất thiết phải là hiện tại (truy vấn như bình thường, Chọn thời điểm quá khứ cần xem (ví dụ: 2023-12-18 14:00:00), xong Bấm "Execute", Prometheus sẽ trả về giá trị instance vector tại thời điểm bạn vừa chọn)

Ứng dụng: Dùng để hiển thị snapshot (trạng thái hiện tại) của các metric, ví dụ như CPU usage tại thời điểm hiện tại.
Cách viết trong PromQL: Khi bạn truy vấn tên metric đơn giản như: http_requests_total
Hoặc truy vấn dựa vào label node_cpu_seconds_total{mode="idle"}
VD: ảnh dưới trả về số lượng prometheus_http_request_total của mỗi dimension ở lần gather data cuối. Mỗi 1 row là 1 timeseries, tuy cùng metric name nhưng khác nhau label. Các cặp label dùng để xác định các object khác nhau


VD rõ nhất của instance vector là ta viết truy vấn dung lượng ổ đĩa trên window server: wmi_logical_disk_free_bytes


Range vector: a set of time series containing a range of data points over time for each time series (1 set các time series nhưng chưa multiple sample cho mỗi time series)

Range vector là một tập hợp các chuỗi thời gian mà mỗi chuỗi chứa nhiều giá trị dữ liệu trong một khoảng thời gian xác định (ví dụ: 5 phút gần nhất).
Instant vector phù hợp khi bạn muốn giám sát trạng thái hiện tại, trong khi range vector là nền tảng cho mọi phân tích sâu hơn về xu hướng, tốc độ thay đổi (rate) hoặc kiểm tra biến động metric theo thời gian
Ứng dụng: Dùng để phân tích xu hướng, tính toán rate, average, phát hiện anomaly hoặc tổng hợp dữ liệu trên một quãng thời gian.
Cách viết trong PromQL: Bằng cách thêm một đoạn selector dưới dạng [thời lượng] vào metric, ví dụ: http_requests_total[5m]
Lưu ý: range vector không thể tạo graph
Hoặc truy vấn dựa vào label rate(node_cpu_seconds_total{mode="idle"}[5m])
VD: 1m ở đây nghĩa là 1 phút, nếu default scrape_interval là 15 thì trong 1’ sẽ có 4 samples, trong 5’ sẽ có 20 samples

Scalar: a simple numeric floating point value (ko có label). VD: 15.21
String: string value (ít khi dùng)

- Selector và matcher
Khi query Prometheus có thể sẽ trả nhiều time series cho cùng 1 metric name với khác label khác nhau, để thu hẹp kết quả trả về dựa theo label, ta có thể dùng selector và matcher
VD: process_cpu_seconds_total{job=’node_exporter’, instance=’localhost:9100’} -> filter theo label job=node_exporte và instance=localhost:9100 . Hoặc có thể chỉ filter theo 1 label

- Có 4 kiểu matcher:
Quality matcher (=): match chính xác label, lưu ý kiểu này có tính đến case sensitive
Negative equality matcher (!=): match các label khác với chỉ định
Regular expression matcher (=~): select label theo dạng regex. VD: prometheus_http_requests_total{handler=~”/api.*”} -> trả về tất cả các timeseries có label là handler bắt đầu bằng /api
Negative regular expression matcher (!~): không match label theo dạng regex

- Operators: có 2 loại operator là binary operators và aggregation operators
Binary operators là operator tính toán dựa trên 2 operands (toán tử). Có 3 loại binary operator:
+ arithmetic binary operator (phép tính): là các phép tính +, -, *, /, %, ^. Được dùng giữa scalar/scalar, vector/scalar, vector/vector. VD: node_memory_MemTotal_bytes/1024/1024
+ comparison binary operator:  là các phép so sánh ==, !=, >, <, >=, <=. Được dùng giữa scalar/scalar, vector/scalar, vector/vector. VD: node_cpu_seconds_total > 500 -> thường được dùng để làm cảnh báo, VD khi CPU vượt quá 500
+ logical/set binary operator: là các phép logic and, or, unless. Chỉ được dùng giữa instant vectors. 
And: label của 2 instant vector phải giống y hệt nhau thì mới trả về giá trị.  (lấy vector1 làm mốc). VD: vector1 and vector2-> nếu có timeseries 1 nào có label của giống y hệt labels của timeseries 2 thì kết quả sẽ trả về timeseries này
Or: returns non-matching element of both left and right side + matching element from left side ( có thể xem như là union của cả 2 vector). 
Note: khi dùng logical/set binary operator có thể kết hợp thêm ignore và on (ingnore để loại trừ 1 label, on để chọn label thuộc 1 tập các label cho trước) . 

VD: prometheus_http_requests_total and ignoring(handler) promhttp_metric_handler_requests_total -> ignore label handler
VD: prometheus_http_requests_total and on(code) promhttp_metric_handler_requests_total -> chỉ match với những time series có trùng 
‘code’ label

Lưu ý khi dùng or/and ta không cần chỉ định key, với ignore và on ta cần chỉ định key. Cả 2 case đều ko cần chỉ định label

Mai phải học lại chỗ này, làm lab
aggregation operators: dùng để combine elements của 1 single instant vector thành 1 vector mới có ít elements hơn với aggregated value.

Chatgpt: Aggregation operators trong Prometheus là các công cụ quan trọng giúp bạn tổng hợp và xử lý các chuỗi số liệu thời gian (time series) thành một vector mới, thường là với kích thước nhỏ hơn. Các operator này thực hiện các phép toán như tổng (sum), giá trị nhỏ nhất (min), lớn nhất (max), trung bình (avg) trên các chuỗi số liệu đầu vào.
Cách hoạt động cơ bản của aggregation operators là lấy một instant vector chứa nhiều time series và tính toán kết quả trên toàn bộ hoặc nhóm các time series theo nhãn (label) cụ thể. Ví dụ, với metric http_requests có nhiều nhãn như method, path, bạn có thể tính tổng số request hoặc trung bình request trên tất cả các nhãn hoặc nhóm theo 1 nhãn cụ thể.

Các aggregation operator type:

VD: sum(prometheus_http_requests_total) : tính tổng http request
       sum(prometheus_http_requests_total) by (code) : tính tổng http request theo từng loại code label
      topk(3, sum(node_cpu_seconds_total) by (mode)): lấy ra 3 mode cpu có time cao nhất (bottomk tương tự)
Max: tìm giá trị lớn nhất
Count: tổng số elemment trong vector (có vẻ như là tổng số timeseries- cần check lại)

- Function
Rate và irate: 

Rate: calculates the per-second average rate of increase of the time series in the range vector (nói đơn giản hơn là tính ra tốc độ mà 1 bộ đếm nào đấy đang tăng). VD nếu ta ta query metric prometheus_http_requests_total thì sẽ nhận được giá trị số lượng http request gần nhất, giá trị đấy ko cung cấp nhiều thông tin lắm do số lượng http request chỉ có tăng chứ không giảm -> cần dùng rate để biết được tốc độ tăng http_request là nhanh hay chậm

		
VD: rate(prometheus_http_requests_total[1m]) -> tính ra trong 1 giây có bao nhiêu http request, với range là hiện tại đến 1’ trước
Note: rate chỉ dùng cho range vector và output ra instant vector. VD query prometheus_http_requests_total{handler=~”/api.*”}[1m] là invalid ( do thêm 1m sẽ thành range vector) mà Prometheus chỉ có thể graph instant vector . Ta thêm rate() vào thì valid 
Rate dùng để tạo graph của slow moving counter

Irate: calculate the instant rate of increase of the timeseries in the range vector. Basically, irate calculates the rate based on the last 2 data points gathered
Irate dùng để tạo graph của fast moving counter
Changes: dùng tính số lượng thay đổi trong 1 range vector
VD: changes(process_start_time_second{job=’node_exporter’}[1h]
-> tính xem trong 1 giờ process restart bao nhiêu lần

Còn thiếu vài chap ở đây nữa, bổ sung sau

Metric type

Metric có các kiểu: 
counter (tăng dần theo thời gian, có thể reset về 0. Counter thường được dùng để track how often a particulat code path được thực thi. VD số lượng request, số lượng task hoàn thành, số lượng lỗi). Counter chỉ có 1 method là inc(), default là tăng 1 đơn vị, có thể custom
gauge (có thể tăng giảm theo thời gian). VD metric về nhiệt độ, current RAM, số lượng thread đang chạy. Gaguge có 3 methods inc(), dec(), set(), default là tăng giảm hoặc set 1 đơn vị, có thể custom
Summary: Một metric loại summary sẽ lấy các samples (mẫu dữ liệu) từ những observations (giá trị quan sát được) để tổng hợp lại. Observations có thể là request duration (thời gian app phản hồi cho request), latency , request size,... Summary track size và number của events. Summary có 1 method là observer(), ta pass size của event làm para cho method này. Summary expose multiple timeseries during a scrape:
Total sum (<basename> _sum) of all observed values
Count (<basename> _count) of events that have been observed
	Summary metric có thể include quatiles over a sliding time window
Histogram: histogram samples observations (có thể là request duration or response size,..) và counts them in configurable buckets. Instrumentation cho histogram giống với summary. Histogram expose multiple timeseries during a scrape:
Total sum (<basename> _sum) of all observed values
Count (<basename> _count) of events that have been observed
Mục đích chính của histogram là tính toán quantiles 
