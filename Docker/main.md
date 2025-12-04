

`docker run --rm -v `pwd`:/app --workdir="/app" maven:3.5.3-jdk8-alpine mvn install -DskipTests=true`
Lệnh này dùng để chạy một container Docker nhằm thực thi quá trình build một dự án Maven trong thư mục hiện tại của hệ điều hành host. Cụ thể:

docker run: Khởi chạy một container mới.

--rm: Tự động xóa container sau khi lệnh bên trong container kết thúc.

-v \pwd`:/app: Gắn thư mục hiện tại (pwdlà lệnh lấy đường dẫn thư mục hiện tại trên Linux) vào bên trong container tại đường dẫn/app`. Code  để build không được copy vào container mà được ánh xạ trực tiếp từ thư mục hiện tại trên máy chủ vào thư mục /app trong container

--workdir="/app": Đặt thư mục làm việc bên trong container là /app.

maven:3.5.3-jdk8-alpine: Sử dụng image Docker của Maven phiên bản 3.5.3 với JDK 8 trên Alpine Linux (hình ảnh nhẹ).

mvn install -DskipTests=true: Thực thi lệnh Maven install để biên dịch và đóng gói dự án, trong đó tham số -DskipTests=true chỉ thị Maven bỏ qua bước chạy các tests.

---

daemon của Docker (dockerd) chạy với quyền root trên hệ thống” nghĩa là tiến trình chính của Docker (dockerd) được khởi động bởi user root và hoạt động với toàn quyền truy cập hệ thống giống như tài khoản root trên Linux. Điều này có nghĩa là dockerd có thể thao tác, sửa đổi file hệ thống, network, cấp phát resource, v.v... không bị giới hạn bởi cơ chế bảo vệ quyền Linux thông thường.

Bạn có thể chứng minh điều này rất dễ dàng bằng cách:

Chạy lệnh kiểm tra các process hệ thống như sau:

bash
ps aux | grep dockerd
Kết quả trả về sẽ thấy tiến trình dockerd thuộc user root (cột đầu của dòng kết quả sẽ có chữ “root”).

Ngoài ra, Docker daemon mặc định sử dụng Unix domain socket /var/run/docker.sock mà chỉ user root hoặc những user thuộc group docker mới có quyền ghi/đọc. Nếu không phải root/user trong group docker, bạn sẽ không giao tiếp được với daemon này


Unix domain socket là một cơ chế giao tiếp giữa các tiến trình (process) trên cùng một máy tính, khác với socket network (TCP/IP) vốn được dùng để giao tiếp giữa các máy qua mạng. Khi nói “Docker daemon mặc định sử dụng Unix domain socket /var/run/docker.sock”, nghĩa là Docker daemon (dockerd) sẽ tạo ra một “điểm giao tiếp” đặc biệt tại file “/var/run/docker.sock”.

Socket này hoạt động giống như một cổng/cầu nối để các chương trình (thường là docker CLI hoặc ứng dụng khác) gửi lệnh/quản lý Docker thông qua việc đọc/ghi dữ liệu với file socket đó, thay vì bằng giao tiếp mạng.

File /var/run/docker.sock hoạt động giống như một file “đặc biệt” trong hệ thống Linux: khi ứng dụng ghi dữ liệu vào đây, dữ liệu đó sẽ tới Docker daemon. Tương tự, daemon gửi phản hồi lại qua socket này.

Chỉ các tiến trình nội bộ trên máy đó mới truy cập được Unix socket, giúp đảm bảo an toàn và hiệu năng cao hơn khi trao đổi dữ liệu cục bộ


Khi bạn gõ lệnh docker trên terminal (ví dụ: docker run, docker ps,...), thực chất bạn đang sử dụng Docker client để giao tiếp với Docker daemon (dockerd) thông qua file Unix domain socket là /var/run/docker.sock.

Docker client sẽ gửi các lệnh, yêu cầu điều khiển container… qua socket này.

Docker daemon ở phía server sẽ nhận, xử lý lệnh và trả kết quả lại cho client cũng thông qua socket đó.

Minh chứng thực tế (dùng công cụ strace phân tích hệ thống) sẽ thấy mỗi khi lệnh docker được chạy, nó sẽ “kết nối” (connect) vào file /var/run/docker.sock thay vì giao tiếp qua internet hay mạng LAN.




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

---

Docker-in-Docker (DinD) nghĩa là trong một container Docker bạn chạy một Docker daemon riêng biệt để có thể tạo, quản lý các container khác bên trong nó. Cụ thể, thay vì dùng Docker daemon trên máy chủ chủ (host), DinD tạo một môi trường Docker độc lập bên trong container và khởi tạo các container con bên trong. Các container con này là con cháu của container cha, nằm trong phạm vi của daemon Docker được khởi động trong container cha, không can thiệp đến Docker host bên ngoài.

Hướng dẫn cơ bản để sử dụng Docker-in-Docker (DinD):

- có thể sử dụng image chính thức docker:dind từ Docker Hub. Lệnh sau sẽ khởi động một container chạy Docker daemon bên trong:
```
docker run --privileged --name dind-test -d docker:dind # --privileged cho phép container có quyền cao để chạy Docker daemon.
```
- Để sử dụng Docker bên trong container đó, bạn có thể vào bash của container:

```
docker exec -it dind-test sh
```

Bên trong, bạn có thể chạy các lệnh Docker như bình thường:

```
docker ps
docker run hello-world
```

Docker-in-Docker (DinD) liên quan mật thiết đến CI/CD pipeline vì nó cung cấp môi trường Docker hoàn toàn cách ly để chạy các bước build, test, và deploy container trong pipeline mà không ảnh hưởng đến Docker host chính.

Trong CI/CD pipeline, khi bạn chạy pipeline (ví dụ GitLab CI, Jenkins, hoặc CircleCI), các job cần build hoặc chạy container.  DinD cho phép chạy một Docker daemon riêng bên trong container. Job pipeline sẽ sử dụng DinD như một môi trường Docker độc lập để build image, chạy container test, rồi push image lên registry.

Lợi ích chính của việc dùng Docker-in-Docker (DinD) thay vì dùng trực tiếp Docker daemon của host trong CI/CD pipeline gồm:

- Cách ly môi trường build/test: DinD tạo ra Docker daemon riêng bên trong container, giúp các bước trong pipeline chạy độc lập, không can thiệp hay ảnh hưởng đến Docker daemon của máy chủ host. Điều này đảm bảo môi trường build nhất quán và an toàn hơn, tránh xung đột với các container hoặc tiến trình ngoài pipeline.

- Tính linh hoạt và tái sử dụng: Với DinD, mỗi job CI/CD có thể có Docker daemon riêng, dễ dàng tái tạo hoặc thay thế mà không ảnh hưởng đến host hay các job khác. 

- Dễ dàng triển khai trên các môi trường khác nhau: DinD giúp chuẩn hóa môi trường CI/CD trên các máy chủ hoặc cloud khác nhau mà không phụ thuộc trực tiếp vào cấu hình Docker daemon trên host.

> Đại khái tưởng tượng thay vì trên host có nhiều tiến trình docker job chạy song song thì khi dùng dind các job sẽ được đặt trong từng container riêng biệt

---

Trick để exec vào filesystem của container trong trường hợp container không có bash hoặc shell
- Tìm PID của process đang chạy trong container. VD `ps -aux | grep kube-apiserver`
- truy cập vào filesystem bằng PID. VD `ls /proc/<PID>/root/`
- Bonus: tìm 1 binary trong file system VD `find /proc/<PID>/root/ | grep kube-apiserver`
