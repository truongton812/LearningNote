

`docker run --rm -v `pwd`:/app --workdir="/app" maven:3.5.3-jdk8-alpine mvn install -DskipTests=true`
Lệnh này dùng để chạy một container Docker nhằm thực thi quá trình build một dự án Maven trong thư mục hiện tại của hệ điều hành host. Cụ thể:

docker run: Khởi chạy một container mới.

--rm: Tự động xóa container sau khi lệnh bên trong container kết thúc.

-v \pwd`:/app: Gắn thư mục hiện tại (pwdlà lệnh lấy đường dẫn thư mục hiện tại trên Linux) vào bên trong container tại đường dẫn/app`. Code  để build không được copy vào container mà được ánh xạ trực tiếp từ thư mục hiện tại trên máy chủ vào thư mục /app trong container

--workdir="/app": Đặt thư mục làm việc bên trong container là /app.

maven:3.5.3-jdk8-alpine: Sử dụng image Docker của Maven phiên bản 3.5.3 với JDK 8 trên Alpine Linux (hình ảnh nhẹ).

mvn install -DskipTests=true: Thực thi lệnh Maven install để biên dịch và đóng gói dự án, trong đó tham số -DskipTests=true chỉ thị Maven bỏ qua bước chạy các tests.


---

Có, việc sử dụng file docker.sock để giao tiếp với dockerd là hoàn toàn khả thi và là cách phổ biến khi các ứng dụng hoặc công cụ muốn tương tác trực tiếp với daemon Docker mà không cần dùng lệnh CLI docker.

Giải thích chi tiết:

dockerd là daemon của Docker, nó lắng nghe các yêu cầu qua socket để quản lý container, network, volume, ...

File docker.sock thường nằm ở /var/run/docker.sock trên hệ thống Linux, đó thực chất là một Unix domain socket.

Khi bạn muốn một ứng dụng giao tiếp với Docker daemon, ứng dụng đó có thể kết nối trực tiếp đến socket này và gửi các yêu cầu API Docker (REST API qua Unix socket).

Các lệnh docker CLI thực chất cũng giao tiếp với daemon Docker qua file này nên bạn hoàn toàn có thể bypass CLI và sử dụng socket này trực tiếp khi cần.

Ví dụ:

Bạn có thể dùng thư viện Docker SDK (cho Python, Go, ...) kết nối trực tiếp đến unix:///var/run/docker.sock.

Hoặc khi viết ứng dụng, bạn gửi các HTTP request đến socket này cũng được (HTTP over Unix socket).

Đây cũng là cách mà các tool như Portainer, cAdvisor, hoặc các plugin giám sát dùng để lấy dữ liệu hay điều khiển Docker.


Example

1. Sử dụng curl để gửi HTTP request qua Unix socket
Docker daemon cung cấp API REST qua Unix socket, vì vậy bạn có thể dùng curl với tùy chọn --unix-socket để gửi request.

Ví dụ lấy thông tin version Docker:

```curl --unix-socket /var/run/docker.sock http://localhost/version```

Ví dụ lấy danh sách container đang chạy:

```curl --unix-socket /var/run/docker.sock http://localhost/containers/json```


2. Ví dụ gửi request POST tạo container
Bạn có thể gửi JSON data để tạo container mới:

bash
curl -X POST --unix-socket /var/run/docker.sock -H "Content-Type: application/json" \
    -d '{"Image": "busybox", "Cmd": ["echo", "hello from docker socket"]}' \
    http://localhost/containers/create
3. Dùng Python với thư viện requests_unixsocket
Bạn cài thư viện hỗ trợ HTTP qua Unix socket:

bash
pip install requests requests_unixsocket
Mã ví dụ lấy thông tin Docker version:

python
import requests_unixsocket

session = requests_unixsocket.Session()
response = session.get('http+unix://%2Fvar%2Frun%2Fdocker.sock/version')

print(response.json())
Giải thích:

http+unix://%2Fvar%2Frun%2Fdocker.sock là cách mã hóa path Unix socket trong URL (dấu / được thay thành %2F).

4. Một số endpoint Docker API phổ biến qua socket
/version: lấy thông tin version Docker daemon

/containers/json: danh sách container

/containers/create: tạo container mới (POST với JSON)

/containers/{id}/start: khởi động container (POST)

/images/json: danh sách image

Bạn có thể tham khảo tài liệu Docker Engine API để biết thêm chi tiết các endpoint:

https://docs.docker.com/engine/api/v1.41/
