# Nginx

Cấu trúc file cấu hình nginx.conf

<img src="1.png">

- main: Khối (context) ngoài cùng, chứa các chỉ thị chung cho toàn bộ Nginx (như user, worker_processes, pid...).
- event: Là khối con nằm trong main context, quy định cách Nginx xử lý các kết nối mạng (như worker_connections...).
- HTTP: Khối lớn chứa các chỉ thị xử lý các yêu cầu HTTP; bên trong có thể có nhiều server block.
- server: Mỗi khối server là đại diện cho một website hoặc một cổng lắng nghe (virtual host). Một khối HTTP có thể chứa nhiều server.
- location: Nằm trong server, dùng để quy định các quy tắc cho từng đường dẫn cụ thể hoặc kiểu file. Một server có thể có nhiều khối location, quy định chi tiết cách Nginx xử lý request tương ứng với từng URL hoặc điều kiện.

Trong thực tế triển khai, thông thường không đưa trực tiếp các block server vào file nginx.conf. Thay vào đó, nên tạo các file cấu hình riêng biệt cho mỗi server block (mỗi domain/website), rồi sử dụng chỉ thị include để nạp các file này vào.
Cấu trúc chuẩn: 
- Trên Ubuntu/Debian, server block được đặt trong thư mục /etc/nginx/sites-available/, sau đó tạo symlink sang /etc/nginx/sites-enabled/. Import vào Nginx thông qua chỉ thị include /etc/nginx/sites-enabled/*
- Trên CentOS server block được đặt trong thư mục /etc/nginx/conf.d/, mỗi file tương ứng cho 1 site và được import vào Nginx thông qua chỉ thị include /etc/nginx/conf.d/*.conf trong nginx.conf.

Ví dụ mẫu:

File nginx.conf (đường dẫn: /etc/nginx/nginx.conf)

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    # Include all server blocks trong thư mục conf.d/
    include /etc/nginx/conf.d/*.conf;
}
```
File server block riêng (ví dụ: /etc/nginx/conf.d/example.com.conf)
```
server {
    listen 80;
    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
