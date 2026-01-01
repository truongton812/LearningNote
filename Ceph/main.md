Ceph có 4 thành phần daemon chính: MON, OSD, MDS, MGR, tạo thành control plane và data plane phân tán.

<img width="1401" height="1080" alt="image" src="https://github.com/user-attachments/assets/3343686d-63c1-4dbe-b4ff-831ed2735575" />​

Ceph Monitor (MON): MON duy trì cluster map (bản đồ OSD, PG, CRUSH), quản lý quorum (ít nhất 3 MON cho HA), xác thực daemon/client. Chúng đồng bộ qua Paxos, client query MON để biết vị trí dữ liệu.
​

Ceph OSD Daemon: OSD lưu trữ dữ liệu thực tế trên disk (1 OSD = 1 disk), xử lý read/write, replication/recovery, heartbeat với MON/MGR. Số lượng OSD quyết định capacity/performance, dùng erasure coding tiết kiệm dung lượng.
​

Ceph Metadata Server (MDS): MDS chỉ cần cho CephFS (file system), quản lý metadata (inode, directory, permission), cache để tránh tải OSD. Không dùng CephFS (object/block) thì không deploy MDS.
​

Ceph Manager (MGR):MGR thu thập metrics, cung cấp dashboard web, REST API, orchestrator (cephadm) tự động deploy daemon. Ít nhất 2 MGR cho HA, tích hợp Prometheus exporter


Bootstrap Host (Host 1)
Khởi động cluster, chạy 3 container: Ceph Monitor (MON daemon) quản lý trạng thái cluster và heartbeat, Ceph Manager (MGR daemon) giám sát metrics/cân bằng, Ceph CLI/Dashboard để quản trị qua giao diện.
​

Host 2-3 (Monitor/Manager Node)
Chạy Ceph Monitor và Ceph Manager dự phòng (HA quorum), Node exporter cho Prometheus monitor. Không lưu dữ liệu, chỉ điều phối.
​

Host 4 (OSD Node)
Ceph OSD daemon (3 container = 3 OSD) lưu dữ liệu thực tế, xử lý replication/recovery. Node exporter export metrics node. Đây là storage backend thực.
​

Ceph Orchestrator
Thành phần quản lý tập trung, chạy trên bootstrap host, điều phối deploy/scale các daemon qua SSH/API tới các host khác mà không cần cài đặt thủ công. Phù hợp Kubernetes (Rook operator) hoặc ceph-ansible.
​

Kiến trúc này tách rõ control plane (Monitor/Manager) từ data plane (OSD), giống pod control-plane vs worker nodes trên K8s bạn quản lý.
