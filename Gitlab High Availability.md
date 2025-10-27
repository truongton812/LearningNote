Các thành phần để tạo cụm gitlab HA

## 1. Cluster Coordination & State Management

Consul (DCS) Distributed Configuration Store: Lưu trữ cấu hình, quản lý leader election cho Patroni, và cung cấp service discovery cho các dịch vụ khác.

- Patroni dùng Consul để lưu metadata về trạng thái PostgreSQL (master / standby).

- Khi node master gặp sự cố, Patroni dựa vào thông tin trong Consul để thực hiện leader election và kích hoạt failover.

- Các dịch vụ khác (như GitLab Rails, PgBouncer, Praefect) truy vấn Consul để service discovery, bảo đảm luôn kết nối đến node và endpoint đang hoạt động.

Có thể thay Consul bằng etcd trong hệ thống GitLab HA, đặc biệt là khi dùng Patroni để quản lý PostgreSQL cluster. Patroni hỗ trợ nhiều loại Distributed Configuration Store (DCS), bao gồm Consul, etcd, ZooKeeper và Kubernetes, tuy nhiên etcd không có sẵn các tính năng service discovery và health check mạnh như Consul

## 2. Database & Connection Layer

#### PostgreSQL sẽ lưu trữ các loại dữ liệu sau:​

- Thông tin người dùng, nhóm, permission và từng quyền truy cập repository.​

- Metadata về repository: tên, mô tả, visibility, commit history (nhưng không lưu trữ trực tiếp file mã nguồn, phần này được lưu ở Git repository trên Gitaly).​

- Issue, merge request, comment, milestone, label, pipeline và kết quả CI/CD liên quan đến dự án.​

- Cấu hình dự án, webhook, service integration, setting liên quan đến các chức năng mở rộng của GitLab.

#### Patroni + PostgreSQL (DB HA) để đảm bảo cụm PostgreSQL có automatic failover -> Patroni phát hiện leader hỏng (bằng cách định kỳ truy vấn và cập nhật vào DCS (ví dụ như Consul) để phát hiện leader hỏng và thực hiện leader election), nó tự động chọn replica mới làm leader, đảm bảo DB HA mà không cần can thiệp thủ công.

Cơ chế hoạt động của patroni:
- Mỗi node Patroni chạy một agent và tất cả các node đều kết nối tới DCS (Có thể là Consul, etcd, ZooKeeper…).
- Node leader sẽ định kỳ cập nhật (renew) khóa leader vào DCS với một giá trị TTL (Time To Live).​
- Các node standby cũng liên tục theo dõi khóa leader trên DCS thông qua polling (định kỳ truy vấn hoặc sử dụng watch nếu được hỗ trợ).
- Nếu leader không cập nhật khóa trong thời gian TTL, các standby sẽ phát hiện khóa đã hết hạn và bắt đầu một vòng leader election mới để chọn node mới làm primary

#### PgBouncer Connection pooler, tạo endpoint ổn định cho ứng dụng và Praefect. 
- Toàn bộ ứng dụng GitLab (Rails, Praefect) không kết nối trực tiếp tới PostgreSQL, mà thông qua PgBouncer.
- PgBouncer quản lý connection pool, giảm tải kết nối và đảm bảo ứng dụng luôn kết nối đến leader hiện tại mà không phải restart.
- Trong quá trình failover, PgBouncer chỉ cần cập nhật thông tin backend, không làm ngắt luồng của người dùng.

## 3. Caching & Queueing Layer

Redis + Sentinel (Cache/Queues HA) Cung cấp cache và hàng đợi HA cho GitLab Rails và Sidekiq.

Redis chứa session, background queues.

Bộ đôi Redis Master + Replica + Sentinel đảm bảo nếu master chết, Sentinel tự động promote replica mới. Sentinel sẽ liên tục giám sát trạng thái Redis master. Nếu master bị hỏng, Sentinel tự động bầu chọn replica tốt nhất để nâng cấp thành master mới và cập nhật lại cấu hình toàn bộ cluster.

Các dịch vụ tiêu thụ (như Sidekiq) truy vấn thông qua endpoint Sentinel, nên không phải cập nhật thủ công khi failover.

## 4. Git Storage Layer

Gitaly Cluster (3 backends) + Praefect (Orchestrator) Quản lý nhân bản repo, điều phối đọc/ghi, loại bỏ SPOF trong lớp Gitaly.

Mỗi repository được lưu trên 3 node Gitaly để bảo toàn dữ liệu.

Praefect hoạt động như orchestrator, điều phối các lệnh đọc/ghi đến backend Gitaly phù hợp.

Khi một backend Gitaly down, Praefect tự động chuyển hướng truy cập sang node khác có bản sao tương ứng.

Nhờ cơ chế này, các lệnh git clone, push, fetch không bị gián đoạn.

## 5. Application Layer
   
GitLab Rails / Workhorse / Puma / Sidekiq chạy trên cả 3 node xử lý web UI, API, và background jobs.

Load được phân phối giữa các node, cho phép quy mô mở rộng ngang.

Workhorse nhận HTTP request đầu vào (UI và API), giao tiếp với Rails, Gitaly và Redis.

Sidekiq trên nhiều node xử lý background jobs song song (ví dụ: CI/CD pipelines, email, indexing).

Nếu một node xuống, các node khác vẫn phục vụ bình thường nhờ load balancer.

## 6. Access & Load Balancing Layer
   
HAProxy + Keepalived (VIP) Load balancer cho HTTP(S), SSH, và (tuỳ chọn) gRPC Praefect. Cung cấp virtual IP, health check, và failover tự động.

HAProxy + Keepalived (VIP) là lớp vào/ra chính cho người dùng.

HAProxy chia tải HTTP(S), SSH, và gRPC traffic giữa các node ứng dụng và Praefect.

Keepalived cung cấp Virtual IP (VIP) chung. Khi một load balancer chết, VIP tự động chuyển sang node dự phòng.

Nhờ health check liên tục, HAProxy chỉ gửi request đến các backend khỏe mạnh.

Phân biệt HAProxy / Keepalived và PGbouncer

|HAProxy|KeepAlived|PGbouncer|
|---|---|---|
|HAProxy là load balancer đa năng cho application layer, chia tải và chuyển tiếp các giao thức TCP/HTTP chứ không tối ưu cho pool database connection.​|Keepalived chỉ cấp VIP và failover lối vào, không trực tiếp điều phối hoặc chuyển tiếp traffic.|PgBouncer chỉ dành cho kết nối PostgreSQL, quản lý pool kết nối dựa trên session/query, giải quyết vấn đề tài nguyên cho database, không load balance traffic web/app.​|

## 7. Object Storage Layer
   
Object Storage (S3 / GCS được khuyến nghị) Lưu trữ artifacts, uploads, LFS, packages, registry. Giúp tách biệt dữ liệu khỏi ổ đĩa local, tăng khả năng mở rộng.

S3 / GCS (hoặc MinIO self-hosted) lưu trữ các dữ liệu không thuộc repository:

Artifacts, uploads, LFS, container registry, packages.

Vì lưu trữ độc lập với node GitLab, Object Storage giúp scale dễ dàng và loại bỏ phụ thuộc đĩa cục bộ.

Khi triển khai multi-zone hoặc multi-region, dữ liệu vẫn truy cập được qua endpoint S3.

## 8. Observability & Backup Strategy
   
Prometheus + Grafana + Exporters Theo dõi hiệu năng, tài nguyên, và cảnh báo.

Backup chiến lược: WAL, Database, Repo, và Object Storage lifecycle.

## Tóm tắt luồng hoạt động toàn cụm

<img width="2048" height="2048" alt="image" src="https://github.com/user-attachments/assets/514d00bd-b145-401d-8a27-33582e6aeb5b" />

Người dùng gửi request (HTTP/SSH/gRPC) → HAProxy VIP → chuyển đến node ứng dụng.

Node ứng dụng (Rails/Workhorse) kiểm tra cache từ Redis, nếu cần truy cập dữ liệu → PgBouncer → PostgreSQL leader.

Các thao tác với repo được xử lý qua Praefect → Gitaly Cluster (backend).

Artifacts và tệp định kèm được ghi lên Object Storage.

Các background jobs chạy qua Sidekiq + Redis queues, đồng thời các chỉ số được thu thập bởi Prometheus.

Tham kháo thêm
https://devops.vn/posts/trien-khai-gitlab-high-availability-production-bang-ansible/
