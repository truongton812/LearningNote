

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

# Basic term
- Sample: là 1 điểm dữ liệu, tức là 1 giá trị tại 1 thời điểm cụ thể của metric

  Ví dụ định dạng một sample: `http_requests_total{method="POST", handler="/login"} 154` nghĩa là tại thời điểm Prometheus thu thập, giá trị của http_requests_total{method="POST", handler="/login"} là 154.
- Timeseries: là một chuỗi các điểm dữ liệu (sample) được ghi nhận liên tiếp qua các mốc thời gian. Mỗi timeseries trong Prometheus được xác định duy nhất bởi:
	- Tên metric (ví dụ: http_requests_total)
	- Một tập các label (cặp key-value, ví dụ: method="POST", handler="/login")
   
  Ví dụ định dạng một timeseries: `http_requests_total{method="POST", handler="/login"}`. 

- Metric: là các chỉ số/quy mô (số liệu) dùng để đo lường trạng thái, hiệu năng, hay các thuộc tính cụ thể của hệ thống hoặc ứng dụng tại một thời điểm. Ví dụ như CPU usage, số lượng request, dung lượng RAM, v.v. Mỗi metric thường có một tên rõ ràng, thể hiện ý nghĩa đo lường, ví dụ: http_requests_total, node_cpu_seconds_total,... Mỗi metric là một tập hợp của nhiều timeseries. Mỗi timeseries sẽ đại diện cho 1 object cần giám sát

  Ví dụ: Metric http_requests_total với các label như method, status, instance sẽ tạo ra nhiều timeseries khác nhau, ví dụ:
	- http_requests_total{method="GET", status="200", instance="web1"}
	- http_requests_total{method="POST", status="500", instance="web2"}

- Target: là server, device,... cần monitor
- Instance: là endpoint cần thu monitor. Ex: 1.2.3.4:5670, 1.2.3.4:5671
- Job: là group của targets hoặc intances có điểm giống nhau
- Exporters: Nếu 1 application muốn đc monitor bởi prometheus, cần add instrumentation vào application code thông qua Prometheus client library. Tuy nhiên nếu ta ko có quyền chỉnh sửa code hoặc cần monitor 1 server -> sử dụng exporter. Nhiệm vụ của exporter là thu thập data theo chỉ định của Prometheus, transform thành định dạng Prometheus cần và gửi về. Có nhiều loại exporter, VD node exporter cho Linux machine (để thu thập metric như CPU, memory, disk space, disk I/O, network bandwidth,...), WM exporter cho Window


# PromQL

### PromQL data type
Trong PromQL (Prometheus Query Language), có 4 kiểu dữ liệu cơ bản:
- Instance vector: Tập hợp nhiều time series, mỗi time series chỉ lấy giá trị tại một thời điểm cụ thể, giá trị này là giá trị gần nhất thời điểm truy vấn mà được lưu trong timeseries database. Đây là kiểu dữ liệu trả về khi truy vấn tên metric hoặc filter bằng label, ví dụ: `wmi_logical_disk_free_bytes #trả về disk space của từng ổ đĩa trên window` hoặc `node_cpu_seconds_total{mode="idle"}`

  Lưu ý là instance vector có thể truy vấn được thời gian là 1 thời điểm cụ thể, không nhất thiết phải là hiện tại. Cách thực hiện: truy vấn tên metric, sau đó chọn thời điểm quá khứ cần xem (ví dụ: 2023-12-18 14:00:00), xong Bấm "Execute", Prometheus sẽ trả về giá trị instance vector tại thời điểm bạn vừa chọn

  Ứng dụng: Dùng để hiển thị snapshot (trạng thái hiện tại) của các metric, ví dụ như CPU usage tại thời điểm hiện tại.

- Range vector: là tập hợp các time series mà trong đó mỗi time series chứa nhiều giá trị dữ liệu trong một khoảng thời gian xác định (ví dụ: 5 phút gần nhất). Đây là kiểu dữ liệu trả về khi truy vấn tên metric metric hoặc filter bằng label và kèm theo [thời lượng] vào metric , ví dụ `http_requests_total[5m]` hoặc `node_cpu_seconds_total{mode="idle"}[5m]` . Nếu scrape_interval là 15s thì kết quả trả về mỗi time series sẽ có 20 samples
  
  Ứng dụng: Dùng để phân tích xu hướng, tính toán rate, average, phát hiện anomaly hoặc tổng hợp dữ liệu trên một quãng thời gian.
  
  Lưu ý: Ta không thể tạo graph của range vector

- Scalar: một giá trị số thực (floating point) đơn lẻ tại một thời điểm, thường dùng trong biểu thức điều kiện hoặc phép tính.
- String: một chuỗi ký tự, ít sử dụng trong kết quả truy vấn thông thường của PromQL, chủ yếu sử dụng làm label, hoặc hiển thị metadata, ví dụ như giá trị label (job="api").


### Selector và Matcher

Khi query Prometheus có thể sẽ trả nhiều time series cho cùng 1 metric name với các labels khác nhau, để thu hẹp kết quả trả về dựa theo label, ta có thể dùng selector và matcher

##### Selector

Selector là thành phần trong PromQL dùng để chọn ra tập hợp time series từ một metric cụ thể. Có hai loại selector phổ biến:
- Instant vector selector: Chọn ra giá trị mới nhất của từng time series tại một thời điểm duy nhất. Ví dụ: http_requests_total
- Range vector selector: Chọn ra nhiều giá trị của từng time series trong một khoảng thời gian. Ví dụ: http_requests_total[5m]

Kết luận: Selector chỉ định “Lấy metric nào về”.

##### Matcher

Matcher là điều kiện (bộ lọc) áp dụng trên label của time series trong selector để chỉ lấy ra các chuỗi dữ liệu phù hợp. Matcher thường đi kèm trong dấu ngoặc nhọn {} và có 4 loại:

- Quality matcher (=): match chính xác label, lưu ý kiểu này có tính đến case sensitive

VD: `process_cpu_seconds_total{job='node_exporter', instance='localhost:9100'}` 
-> kết quả trả về giá trị hiện tại (tức thời điểm bạn truy vấn) của tất cả các time series từ metric process_cpu_seconds_total mà có cả 2 nhãn `job='node_exporter'` và `instance='localhost:9100'`. Lưu ý là nếu một time series cùng có 2 nhãn này và thêm các nhãn khác nữa, nó vẫn được đưa vào kết quả truy vấn.

Kết quả có thể là:
```
process_cpu_seconds_total{job="node_exporter", instance="localhost:9100", cpu="0"} 1234.5
process_cpu_seconds_total{job="node_exporter", instance="localhost:9100", cpu="1"} 5678.9
```

- Negative equality matcher (!=): match các label khác với chỉ định
- Regular expression matcher (&#61;&#126;): select label theo dạng regex. VD: `prometheus_http_requests_total{handler=~”/api.*”}` -> trả về tất cả các timeseries có label là `handler` bắt đầu bằng `/api`
- Negative regular expression matcher (!~): không match label theo dạng regex

Kết luận: Matcher chỉ định “Lấy metric đó nhưng điều kiện nhãn thế nào”.

### Operators

Là các ký hiệu hoặc từ khóa đại diện cho các phép toán hoặc thao tác xử lý dữ liệu được áp dụng lên metric hoặc tập hợp dữ liệu time series. Operator được dùng để kết hợp, so sánh hoặc tính toán giữa các metric hoặc giá trị, giúp truy vấn dữ liệu theo cách mong muốn. Có 2 loại operator là binary operators và aggregation operators

##### Binary operators 

Là operator tính toán dựa trên 2 operands (toán tử). Có 3 loại binary operator:

- Arithmetic binary operator: là các phép tính +, -, *, /, %, ^. Được dùng giữa vector/scalar, vector/vector, scalar/scalar. VD: `node_memory_MemTotal_bytes/1024/1024`
- Comparison binary operator:  là các phép so sánh ==, !=, >, <, >=, <=. Được dùng giữa vector/scalar, vector/vector, scalar/scalar, thường được dùng để làm cảnh báo, VD khi CPU vượt quá 500 `node_cpu_seconds_total > 500`
- Logical binary operator: là các phép logic and, or, unless. Chỉ được dùng giữa instant vectors. 
	- And: Trả về các time series mà cùng tồn tại (match) ở cả hai vector bên trái và bên phải dựa trên tập nhãn giống nhau. Cách hoạt động: Nếu một time series ở vector bên trái có cùng bộ label với một time series ở vector bên phải, giá trị sẽ được giữ lại ở kết quả. Nếu không có cặp nào khớp label giữa hai bên, time series đó bị loại khỏi kết quả.
   
        VD: `up{job="api"} and http_requests_total{job="api"}`
   
        -> Chỉ những time series của metric up có nhãn job="api" mà đồng thời cũng xuất hiện ở metric http_requests_total (tức khớp nhãn giữa hai bên) mới được hiển thị

	- Or: Kết hợp các time series từ cả hai phía, giữ lại toàn bộ series xuất hiện ở bất kỳ bên nào. Cách hoạt động: Nếu time series chỉ xuất hiện ở một bên, vẫn được đưa vào kết quả. Nếu cùng nhãn xuất hiện ở cả hai metric, giá trị lấy từ bên trái.

        VD: `http_requests_total{status="200"} or http_requests_total{status="500"}`

        -> Kết quả bao gồm cả những time series có status là 200 hoặc 500, không bỏ sót cái nào
   

Khi dùng logical binary operator có thể kết hợp thêm ignoring và on (ingnore để loại trừ 1 label, on để chọn label thuộc 1 tập các label cho trước) . 

- ignoring (\<label list>):
Loại bỏ các nhãn chỉ định trong danh sách ra khỏi quá trình so khớp vector. Nghĩa là tất cả các nhãn trừ những nhãn ignoring sẽ được dùng để so khớp.

  VD1: `prometheus_http_requests_total and ignoring(handler) promhttp_metric_handler_requests_total`

  -> trả về các time series thuộc metric prometheus_http_requests_total mà (ngoại trừ nhãn handler) có bộ nhãn trùng khớp với bất kỳ time series nào của metric promhttp_metric_handler_requests_total. Kết quả là giá trị của metric bên trái – tức chỉ các series từ prometheus_http_requests_total, nhưng chỉ giữ lại những series mà (ngoại trừ nhãn handler) cũng có xuất hiện ở promhttp_metric_handler_requests_total

  VD2: `sum(http_requests_total) by (job, status) + ignoring(status) sum(http_errors_total) by (job, status)`

  -> Phép cộng sẽ thực hiện mà bỏ qua nhãn status, chỉ matching dựa trên các nhãn còn lại
  
- on (\<label list>):
Chỉ dùng các nhãn được chỉ định trong danh sách để ghép các time series giữa hai bên. Những nhãn không nằm trong danh sách sẽ bị bỏ qua khi so khớp.

  VD1: `prometheus_http_requests_total and on(code) promhttp_metric_handler_requests_total`

  -> Kết quả truy vấn là các time series thuộc metric prometheus_http_requests_total mà trường code trùng khớp với ít nhất một time series của metric promhttp_metric_handler_requests_total. Toàn bộ nhãn và giá trị của phía trái được giữ nguyên

  VD2: `sum(http_requests_total) by (method) / on(method) sum(http_response_time_seconds) by (method)`

  -> Biểu thức này chia tổng số requests cho tổng response time, ghép cặp dựa trên nhãn method


##### Aggregation operators

Là các toán tử dùng để tổng hợp và xử lý các time series thành một vector mới, thường là với kích thước nhỏ hơn. Cách hoạt động cơ bản của aggregation operators là lấy một instant vector chứa nhiều time series và tính toán kết quả trên toàn bộ hoặc nhóm các time series theo label cụ thể. Ví dụ, với metric http_requests có nhiều nhãn như method, path, bạn có thể tính tổng số request hoặc trung bình request trên tất cả các nhãn hoặc nhóm theo 1 nhãn cụ thể. 
Lưu ý: Aggregation operators chỉ áp dụng với "instant vector"
      
Các aggregation operator type:
- sum: Tính tổng các giá trị của các chuỗi time series. Ví dụ: `sum(prometheus_http_requests_total) by (code)` -> tính tổng http request theo từng loại code label
- avg: Tính giá trị trung bình. Ví dụ: `avg(cpu_usage)`
- min / max: Lấy giá trị nhỏ nhất/lớn nhất trong nhóm. Ví dụ: `min(node_memory_MemFree_bytes)`, `max(node_memory_MemFree_bytes)`
- count: Đếm số lượng time series. Ví dụ: `count(up)`
- count_values: Đếm số lần xuất hiện của mỗi giá trị cụ thể trong time series và trả về thành từng nhóm theo value.
- stddev, stdvar: Tính độ lệch chuẩn (standard deviation) và phương sai (variance).
- quantile: Tính giá trị percentile mong muốn (ví dụ: median, 95th).
- topk(k, metric): Lấy k giá trị lớn nhất. VD: `topk(3, sum(node_cpu_seconds_total) by (mode))`: lấy ra 3 mode cpu có time cao nhất
- bottomk(k, metric): Lấy k giá trị nhỏ nhất.
- group: Gom nhóm các time series mà không quan tâm đến giá trị, mỗi nhóm luôn trả về 1 (dùng cho các tình huống kiểm tra sự tồn tại).

Cách sử dụng aggregation operator với by và without
- by (label): Gom nhóm theo một hoặc nhiều nhãn; kết quả giữ lại các nhãn này. VD: `sum by(instance) (node_cpu_seconds_total)` -> Cộng dồn cpu theo từng instance
- without (label): Gom nhóm và loại bỏ các nhãn chỉ định khỏi kết quả. VD: `max without(mode) (node_cpu_seconds_total)` -> Lấy giá trị max, loại bỏ nhãn "mode"


### Function

PromQL cung cấp rất nhiều function giúp xử lý, phân tích, và tổng hợp dữ liệu time series. Các function này có thể chia thành các nhóm chính:

1. Hàm tính toán tốc độ, biến động
- rate(): Tính tốc độ tăng trung bình trên giây của một counter trong 1 khoảng thời gian, chỉ áp dụng với metric kiểu counter. Hàm rate giúp ta nắm được xu hướng của metric đang tăng nhanh hay châm, điều mà bản thân counter metric không thể hiện được. VD khi query metric prometheus_http_requests_total thì sẽ nhận được giá trị số lượng http request gần nhất, giá trị đấy không thể hiện được xu hướng do số lượng http request là 1 số luôn tăng -> rate giúp biết được tốc độ tăng http_request là nhanh hay chậm: `rate(prometheus_http_requests_total[1m])` để tính ra trong 1 giây có bao nhiêu http request, với range là hiện tại đến 1’ trước
  Rate() còn giúp tạo graph cho range vector do output của rate() là instance vector, và Prometheus chỉ có thể tạo graph của instance vector
- irate(): Tốc độ tăng tức thời (tính theo hai điểm dữ liệu gần nhất). Irate dùng để tạo graph của fast moving counter
- changes: dùng tính số lần thay đổi của 1 metric, thường chỉ dùng với các metric ít thay đổi. VD: `changes(process_start_time_second{job='node_exporter'}[1h]` -> tính xem trong 1 giờ process "node exporter" restart bao nhiêu lần
- deriv(): dùng để tính xem 1 gauge metric thay đổi nhanh hay chậm. Có thể xem như là hàm rate() áp dụng cho gauge metric. VD: `deriv(process_resident_memory_bytes{job='prometheus'}[1h])` -> tính tốc độ thay đổi bộ nhớ thực tế (bytes trên giây) mà tiến trình Prometheus đang sử dụng trong 1 giờ gần nhất
- predict_linear(): Dự đoán giá trị sau một thời gian dựa trên xu hướng hiện tại. VD: `predict_linear(my_metric[10m], 30)` -> Dự đoán giá trị metric my_metric trong 30 giây tiếp theo dựa trên dữ liệu 10 phút trước đó.
- increase(): Tổng mức tăng của counter trong một khoảng thời gian.
- delta(): Thay đổi tuyệt đối của gauge trong một period nhất định.

2. Hàm tổng hợp, thống kê trên cửa sổ thời gian

Đây bản chất là các aggregation operator tuy nhiên có thể sử dụng cho range vector

- avg_over_time(): Giá trị trung bình trong một khoảng thời gian.
- sum_over_time(): Tổng giá trị.
- min_over_time(), max_over_time(): Giá trị nhỏ nhất/lớn nhất. VD: `max_over_time(node_cpu_seconds_total[1h]}` -> trả về giá trị lớn nhất của từng time series của metric node_cpu_seconds_total trong vòng 1 giờ gần nhất
- count_over_time(): Số lượng sample có trong khoảng thời gian.
- quantile_over_time(): Tính percentile (như median, 95th percentile).
- stddev_over_time(), stdvar_over_time(): Độ lệch chuẩn/phương sai.

3. Hàm xử lý instance vector:
- sort(): sắp xếp các time series theo thứ tự tăng dần của value
- sort_desc(): sắp xếp các time series theo thứ tự giảm dần của value
- sort_by_label()

4. Hàm phân tích histogram
- histogram_quantile(): Tính giá trị percentile (ví dụ P99, P95) từ metric dạng histogram bucket. Ví dụ: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

5. Hàm biến đổi, toán học
- abs(): Giá trị tuyệt đối.
- ceil(), floor(): Làm tròn lên/xuống.
- sqrt(): Căn bậc hai.
- clamp_min(), clamp_max(), clamp(): Giới hạn giá trị nhỏ nhất, lớn nhất, hoặc trong khoảng.
- round(): Làm tròn tới số lẻ đã chỉ định.

6. Hàm thao tác label/metadata
- label_join(), label_replace(): Nối, thay thế các giá trị nhãn (label) thành nhãn mới.
- absent(): Kiểm tra một metric có xuất hiện hay không.
- absent_over_time(): Check metric có vắng mặt liên tục trong một khoảng thời gian.

7. Hàm ngày, giờ
- year(), month(), day_of_month(), days_in_month(), day_of_week(), hour(), minute(), second(), timestamp(), time(),  : Trích xuất thông tin thời gian từ time series hoặc hệ thống.

8. Hàm đặc biệt cho advanced use
- present_over_time(): Trả về 1 nếu có sample ở trong khoảng thời gian.
- resets(): Đếm số lần counter bị reset.
- last_over_time(): Giá trị sample cuối cùng trong khoảng thời gian.

---
Giải thích thêm về cách tạo graph của hàm rate()
Khi bạn query: rate(http_requests_total[3m]) Prometheus sẽ tính tốc độ trung bình trên mỗi giây của metric http_requests_total trong từng cửa sổ 3 phút kết thúc lần lượt tại mỗi điểm sampling 12:00, 12:01, 12:02, ... Và nếu ta set step trong graph là 1 phút (60s) thì kết quả sẽ là:

Cửa sổ tại 12:00 sẽ tính dựa trên dữ liệu từ 11:57 đến 12:00. -> điểm sampling tại thời điểm 12:00 trên đồ thị nhận giá trị này

Cửa sổ tại 12:01 sẽ tính dựa trên dữ liệu từ 11:58 đến 12:01. -> điểm sampling tại thời điểm 12:01 trên đồ thị nhận giá trị này

Cửa sổ tại 12:02 sẽ tính dựa trên dữ liệu từ 11:59 đến 12:02. -> điểm sampling tại thời điểm 12:02 trên đồ thị nhận giá trị này

---


#### Metric type

Metric có các kiểu: 
- counter (tăng dần theo thời gian, có thể reset về 0. Counter thường được dùng để track how often a particulat code path được thực thi. VD số lượng request, số lượng task hoàn thành, số lượng lỗi). Counter chỉ có 1 method là inc(), default là tăng 1 đơn vị, có thể custom
- gauge (có thể tăng giảm theo thời gian). VD metric về nhiệt độ, current RAM, số lượng thread đang chạy. Gaguge có 3 methods inc(), dec(), set(), default là tăng giảm hoặc set 1 đơn vị, có thể custom
- Summary: Một metric loại summary sẽ lấy các samples (mẫu dữ liệu) từ những observations (giá trị quan sát được) để tổng hợp lại. Observations có thể là request duration (thời gian app phản hồi cho request), latency , request size,... Summary track size và number của events. Summary có 1 method là observer(), ta pass size của event làm para cho method này. Summary expose multiple timeseries during a scrape:
	- Total sum (<basename> _sum) of all observed values
	- Count (<basename> _count) of events that have been observed
     Summary metric có thể include quatiles over a sliding time window
- Histogram: histogram samples observations (có thể là request duration or response size,..) và counts them in configurable buckets. Instrumentation cho histogram giống với summary. Histogram expose multiple timeseries during a scrape:
	- Total sum (<basename> _sum) of all observed values
	- Count (<basename> _count) of events that have been observed
      Mục đích chính của histogram là tính toán quantiles 

---

Thư viện prometheus_client trong Python là một client chính thức giúp ứng dụng của bạn xuất các chỉ số (metrics) theo định dạng mà hệ thống giám sát Prometheus hiểu được. Nói cách khác, nó dùng để tích hợp đo lường (instrumentation) trực tiếp vào mã nguồn ứng dụng nhằm tạo ra các metric như counter (bộ đếm), gauge (giá trị đo lường có thể tăng giảm), histogram (thống kê phân phối giá trị) để Prometheus có thể thu thập.

Với prometheus_client bạn có thể:

- Định nghĩa các loại metric để theo dõi các thông số quan trọng (ví dụ: số lượng request HTTP, thời gian xử lý request, số lượng người dùng hiện tại, v.v).

- Khởi động một HTTP server riêng trên một cổng để Prometheus chủ động gửi request thu thập các chỉ số (mô hình pull).

- Tăng hoặc cập nhật các giá trị metric trong ứng dụng một cách dễ dàng chỉ với vài dòng code.

Thư viện này hỗ trợ rất tốt cho việc giám sát hiệu năng, tình trạng hoạt động của ứng dụng, giúp lập trình viên và đội ngũ vận hành dễ dàng nắm bắt các số liệu từ ứng dụng để cảnh báo hoặc phân tích sau này.


