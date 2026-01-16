Trong Grafana Alert Rule, trường “Interval” (còn gọi là evaluation interval hoặc group interval) và time range trong query (ví dụ [10m]) là hai khái niệm khác nhau, nhưng chúng phối hợp với nhau để quyết định khi nào cảnh báo được đánh giá và có cảnh báo hay không.
​
## 1. Interval trong Alert Rule (Evaluation Interval)
- Là tần suất mà Grafana đánh giá lại rule này.
​- Ví dụ: nếu set Interval = 1m, thì mỗi 1 phút Grafana sẽ:
  - Chạy lại query.
  - Kiểm tra điều kiện (condition) có thỏa mãn hay không.
  - Cập nhật trạng thái cảnh báo (Pending → Firing hoặc Firing → Normal).
- Nếu để interval cao thì có thể sẽ bỏ lỡ các sự kiện ngắn. Ví dụ interval là 10 phút thì nếu lỗi xảy ra ngay sau một lần đánh giá (ví dụ: 09:00:01), thì phải đợi đến lần đánh giá tiếp theo (09:10) mới được kiểm tra → trễ gần 10 phút.
​​
## 2. Time range trong query (ví dụ [10m])
- Là khoảng thời gian dữ liệu mà query lấy về để tính toán. Giúp xác định “cửa sổ dữ liệu” để tính metric
- Time range ​không ảnh hưởng đến tần suất đánh giá rule, chỉ ảnh hưởng đến dữ liệu đầu vào của rule.
​- Ví dụ:
```
count_over_time({job="BE03"} |= `responded` |= `HTTP GET /api/` | regexp `responded (?P<status>\d+) in (?P<duration_ms>\d+) ms` | duration_ms > 300 [10m])`
```
[10m] có nghĩa là: lấy log trong 10 phút gần nhất (tính từ thời điểm đánh giá) để đếm số lần request chậm.
​
## 3. Ví dụ cụ thể
Giả sử: Alert Rule có interval = 1m và Condition: count_over_time(...) > 5

Query là 
```
count_over_time({job="BE03"} |= `responded` |= `HTTP GET /api/` | regexp `responded (?P<status>\d+) in (?P<duration_ms>\d+) ms` | duration_ms > 300 [10m])`
```

Kết quả:
- Mỗi 1 phút, Grafana sẽ lấy log trong 10 phút gần nhất (tính từ thời điểm đó).
​- Đếm số lần request chậm > 300 ms trong 10 phút đó.
​- Nếu số lần > 5 → cảnh báo chuyển sang trạng thái Firing (sau khi qua Pending period).
​
​
