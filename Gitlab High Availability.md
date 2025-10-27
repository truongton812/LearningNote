Các thành phần để tạo cụm gitlab HA

1. Cluster Coordination & State Management

Consul (DCS) Distributed Configuration Store: Lưu trữ cấu hình, quản lý leader election cho Patroni, và cung cấp service discovery cho các dịch vụ khác.

2. Database & Connection Layer

Patroni + PostgreSQL (DB HA) Cụm PostgreSQL có automatic failover và synchronous replication.

PgBouncer Connection pooler, tạo endpoint ổn định cho ứng dụng và Praefect.

3. Caching & Queueing Layer

Redis + Sentinel (Cache/Queues HA) Cung cấp cache và hàng đợi HA cho GitLab Rails và Sidekiq.

4. Git Storage Layer

Gitaly Cluster (3 backends) + Praefect (Orchestrator) Quản lý nhân bản repo, điều phối đọc/ghi, loại bỏ SPOF trong lớp Gitaly.

5. Application Layer
   
GitLab Rails / Workhorse / Puma / Sidekiq chạy trên cả 3 node xử lý web UI, API, và background jobs.

6. Access & Load Balancing Layer
   
HAProxy + Keepalived (VIP) Load balancer cho HTTP(S), SSH, và (tuỳ chọn) gRPC Praefect. Cung cấp virtual IP, health check, và failover tự động.

7. Object Storage Layer
   
Object Storage (S3 / GCS được khuyến nghị) Lưu trữ artifacts, uploads, LFS, packages, registry. Giúp tách biệt dữ liệu khỏi ổ đĩa local, tăng khả năng mở rộng.

8. Observability & Backup Strategy
   
Prometheus + Grafana + Exporters Theo dõi hiệu năng, tài nguyên, và cảnh báo.

Backup chiến lược: WAL, Database, Repo, và Object Storage lifecycle.
