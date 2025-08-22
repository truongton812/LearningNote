

`docker run --rm -v `pwd`:/app --workdir="/app" maven:3.5.3-jdk8-alpine mvn install -DskipTests=true`
Lệnh này dùng để chạy một container Docker nhằm thực thi quá trình build một dự án Maven trong thư mục hiện tại của hệ điều hành host. Cụ thể:

docker run: Khởi chạy một container mới.

--rm: Tự động xóa container sau khi lệnh bên trong container kết thúc.

-v \pwd`:/app: Gắn thư mục hiện tại (pwdlà lệnh lấy đường dẫn thư mục hiện tại trên Linux) vào bên trong container tại đường dẫn/app`. Code  để build không được copy vào container mà được ánh xạ trực tiếp từ thư mục hiện tại trên máy chủ vào thư mục /app trong container

--workdir="/app": Đặt thư mục làm việc bên trong container là /app.

maven:3.5.3-jdk8-alpine: Sử dụng image Docker của Maven phiên bản 3.5.3 với JDK 8 trên Alpine Linux (hình ảnh nhẹ).

mvn install -DskipTests=true: Thực thi lệnh Maven install để biên dịch và đóng gói dự án, trong đó tham số -DskipTests=true chỉ thị Maven bỏ qua bước chạy các tests.

