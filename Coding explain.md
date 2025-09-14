

Đoạn code JavaScript sau:

```javascript
axios.defaults.baseURL = process.env.REACT_APP_BASE_API_URL;
```

có ý nghĩa là thiết lập địa chỉ gốc (base URL) mặc định cho tất cả các request HTTP được thực hiện bởi thư viện Axios trong ứng dụng. Cụ thể:

axios.defaults.baseURL: Là thuộc tính cấu hình mặc định của Axios, quy định một địa chỉ URL cơ sở sẽ được tự động thêm tiền tố cho tất cả các request (vd: GET, POST) mà Axios gửi đi. Nhờ đó, khi gọi axios.get('/users'), thực chất URL đầy đủ sẽ là baseURL + '/users'.

process.env.REACT_APP_BASE_API_URL: Đây là biến môi trường trong ReactJS, được định nghĩa trong file cấu hình môi trường .env. Nó thường lưu trữ URL endpoint của API backend mà frontend sẽ truy cập (ví dụ https://api.example.com). Việc dùng biến môi trường giúp dễ dàng thay đổi URL mà không cần sửa code, phù hợp với các môi trường khác nhau (dev, staging, production).

-> Đoạn code này thiết lập axios.defaults.baseURL lấy giá trị từ biến môi trường REACT_APP_BASE_API_URL trong file .env, tức là URL của API backend mà frontend sẽ gọi đến.

Khi frontend sử dụng Axios để gửi các request HTTP (GET, POST, v.v.), Axios sẽ tự động thêm tiền tố URL từ biến này để kết nối đúng đến backend.

---

Eureka là một dịch vụ Service Discovery giúp các dịch vụ trong hệ thống có thể tự động phát hiện và kết nối với nhau mà không cần biết địa chỉ IP hay cổng cụ thể trước. Eureka giống như một DNS nội bộ dành cho microservices, giúp tự động quản lý địa chỉ và trạng thái các dịch vụ trong kiến trúc phân tán.

Hoạt động chính của Eureka gồm:

- Eureka Server (Service Registry): Là nơi tập trung đăng ký thông tin các instance của dịch vụ. Các dịch vụ khi khởi động sẽ đăng ký (register) vào Eureka Server với thông tin như hostname, cổng, metadata.

- Eureka Client: Các dịch vụ microservices sẽ dùng Eureka Client để đăng ký với Eureka Server và cũng để lấy danh sách các dịch vụ khác đã đăng ký nhằm gọi tới chúng dễ dàng.

Ví dụ cấu hình cho Eureka client -> nó sẽ đăng ký service của mình với Eureka Server. 

```
eureka:
  instance:
    hostname: user-service #Thiết lập tên máy chủ (hostname) đại diện cho instance dịch vụ khi đăng ký với Eureka Server
    instance-id: ${spring.application.name} #Định danh duy nhất của instance dịch vụ, dùng giá trị từ biến ${spring.application.name} (tên app)
  client:
    service-url:
      defaultZone: http://discovery-server:8761/eureka/ #Đây là URL của server Eureka để client (ứng dụng này) đăng ký và lấy danh sách service khác. Client sẽ gửi yêu cầu đăng ký dịch vụ đến địa chỉ http://discovery-server:8761/eureka/.
```

---

Spring Boot hỗ trợ cơ chế binding các biến môi trường vào các thuộc tính cấu hình ứng dụng (ví dụ thuộc tính datasource).

Ví dụ biến môi trường SPRING_DATASOURCE_URL sẽ tự động được ánh xạ và gán giá trị cho thuộc tính spring.datasource.url trong cấu hình Spring Boot.

Tương tự, SPRING_DATASOURCE_USERNAME sẽ gán cho spring.datasource.username và SPRING_DATASOURCE_PASSWORD cho spring.datasource.password.

Khi khai báo environment của container trong Docker Compose, các biến môi trường như SPRING_DATASOURCE_URL, SPRING_DATASOURCE_USERNAME, SPRING_DATASOURCE_PASSWORD sẽ override những giá trị spring.datasource.url trong application.yaml
