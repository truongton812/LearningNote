Ceph có 4 thành phần daemon chính: MON, OSD, MDS, MGR, tạo thành control plane và data plane phân tán.

<img width="1401" height="1080" alt="image" src="https://github.com/user-attachments/assets/3343686d-63c1-4dbe-b4ff-831ed2735575" />​

Ceph Monitor (MON): MON lưu thông tin của cluster (bản đồ OSD, MDS, RGW), quản lý các node nào available/unavailable, quản lý quorum (ít nhất 3 MON cho HA), quản lý xác thực daemon/client, giám sát hoạt động của OSD. Chúng đồng bộ qua Paxos, client query MON để biết vị trí dữ liệu.
​

Ceph OSD Daemon: OSD ánh xạ đến disk để lưu trữ dữ liệu(1 OSD = 1 disk), xử lý read/write, replication/recovery, heartbeat với MON/MGR. Số lượng OSD quyết định capacity/performance, dùng erasure coding tiết kiệm dung lượng.
​

Ceph Metadata Server (MDS): MDS chỉ cần cho CephFS (file system), quản lý metadata (inode, directory, permission), cache để tránh tải OSD. Không dùng CephFS (object/block) thì không deploy MDS.
​

Ceph Manager (MGR):MGR thu thập metrics, cung cấp dashboard web, REST API, orchestrator (cephadm) tự động deploy daemon. MGR giám sát và vận hành cluster (MON chỉ lưu giữ thông tin, còn MGR mới là thành phần thực thi). Ít nhất 2 MGR cho HA, tích hợp Prometheus exporter. Khi tương tác với dashboard web đồng nghĩa với tương tác với MGR

---
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

---

Các lệnh làm việc với ceph:
- `ceph orch ls` : liệt kê ra tất cả daemon trên các host (output sẽ giống xem trong Administration > Services trên dashboard)
- `ceph orch ls mon` : liệt kê ra các mon
- `ceph orch ps` : liệt kê ra các container (alertmanager, ceph-exporter, crash - lưu log thì phải, grafana, mgr, mon, node-exporter, prometheus,..) (hay còn gọi là daemon) trên tất cả các node. Thêm option --refresh để lấy mới nhất
- `ceph orch ps` : liệt kê tất cả daemon trên 1 host cụ thể
- `ceph orch apply mon --unmanaged` : không muốn tự động nữa ?? Do khi khởi tạo thì 1 node add vào cluster sẽ có đủ các chức năng, dùng lệnh này để ta tự add chức năng thủ công cho node
- `ceph orch apply mon "3 <hostname1> <hostname2> <hostname3>"`: triển khai mon lên 3 node (??)
- `ceph mon stat`: xác định đồng thuận
- `ceph orch ps --daemon-type mon`: 
- `ceph orch host add <hostname> <ip> --label mon,osd,mds,rgw,_admin`: add host vào ceph cluster với các chức năng mon, osd, mds, rgw và admin (check lại admin có quyền gì) -> sau khi add xong thì trong trang quản trị dashboard của ceph mục "Hosts" sẽ hiển thị các node mới
- `ceph orch host ls` : liệt kê các host quản trị bởi cephadm (xem trên host cài daemon gì)
- `ceph cephadm check-host <hostname>`: đồng bộ giữa các node (??)
- `ceph orch apply mgr 2`: cấu hình 2 mgr
- `ceph mgr stat`
- `ceph orch ps --daemon-type mgr`
- `ceph -s`: lệnh để debug cụm, xem health của cụm
Trong thư mục /etc/ceph/ có file ceph.conf là để cấu hình, ceph.pub là public key để chia sẻ với các node khác, giúp các node khác join vào cluster
- `ceph orch status`: xem trạng thái orchestrator
- `ceph df`: xem disk
---

### OSD
Trong ceph mối OSD đại diện cho 1 ổ disk

Có thể xem các disk trong cụm bằng dashboard, vào "Physical disks"

Gán osd vào cụm (do mặc định các disk chỉ tồn tại chứ chưa map với osd). Lưu ý trên production thì nên format disk trước khi gắn vào osd để tránh lỗi

- `ceph orch device ls --refresh`: xem các ổ hiện có
- tạo file cấu hình
```
service_type: osd
service_id: hdd_osds
placement:
  pattern: 'node*'
data_devices:
  paths: #hoặc dùng bộ lọc theo dung lượng nếu tên ổ đĩa không đồng nhất. Nếu tên ổ và dung lương khác nhau thì phải tạo nhiều file cấu hình (??)
  - /dev/sdb
  - /dev/sdc
  - /dev/sdd
filter_logic: AND
object_store: bluestore
```
- apply bằng lệnh `ceph orch apply -i <file>` -> check trong mục OSD của dashboard sẽ thấy có

---

### Tạo rule / pool
- Pool là gì
- Khi tạo cluster mặc định có pool .mgr
- `ceph osd pool ls`: liệt kê các pool
- `ceph osd crush class ls`: liệt kê loại disk là hdd hay sdd
- `ceph osd pool create test-pool 32 32`: tạo pool tên test-pool (tìm hiểu 32 là gì - lquan đến pg)
- `ceph osd pool set test-pool size 3`: dữ liệu ghi vào sẽ ghi ở 3 nơi (3 copy).
- `ceph osd pool set test-pool min_size 2`: y/c ceph trong trường hợp lỗi thì phải giữ tối thiểu là 2
- Khi tạo pool xong phải enable application lên (cần check lại)
- Muốn xóa pool cần phải tắt cơ chế bảo vệ `ceph config set mon mon_allow_pool_delete true`
- lệnh xóa pool: `ceph osd pool rm test-pool test-pool --yes-i-really-really-mean-it`
- `ceph config set mon mon_allow_pool_delete false`: enable lại cơ chế bảo vệ
<img width="1361" height="932" alt="image" src="https://github.com/user-attachments/assets/c3367d2a-5e7b-4d3c-966d-8b7fe0b12588" />

Tất cả mọi câu lệnh trên đều có thể thực thi trên giao diện

---
Cách ceph hoạt động
- Khi object được lưu vào sẽ được lưu làm 3 bản copy trên 3 node (check lại thông tin)
- Tìm hiểu về broadcast recovery trong ceph làm cho truy cập ceph chậm và cách xử lý bằng CRUSH map

---
### CRUSH rule
- CRUSH rule trong Ceph là tập luật xác định cách dữ liệu được đặt lên các OSD trong toàn bộ cluster, dựa trên cây CRUSH (host, rack, datacenter, class hdd/ssd, v.v.).
- CRUSH (Controlled Replication Under Scalable Hashing) là thuật toán tính toán vị trí lưu PG/object, tránh phải có central metadata server.
​- CRUSH rule là “policy” nói cho CRUSH biết cần chọn OSD nào, ở failure domain nào, bao nhiêu replica/chunk.
​- Ví dụ dùng thực tế:
  - Bạn có thể tạo một rule cho pool replicated 3 bản sao, yêu cầu mỗi bản nằm trên host khác nhau và chỉ dùng class ssd: ceph osd crush rule create-replicated fast default host ssd.
​  - Hoặc rule cho pool erasure-coded, chọn OSD trải đều trên nhiều rack, dùng chế độ indep để xử lý khi OSD bị down.
​- Áp dụng cho pool:
  - Mỗi pool Ceph sẽ gán với một CRUSH rule; rule này quyết định placement và chiến lược replication/erasure coding của toàn bộ PG trong pool đó.
​  - Thay đổi rule (hoặc gán rule khác cho pool) sẽ làm Ceph remap lại PG để đáp ứng chính sách mới, ví dụ dàn lại dữ liệu sang SSD hoặc sang DC khác

Cách thực sử dụng CRUSH rule:
- tạo pool (theo hướng dẫn ở trên): `ceph osd pool create pool-hdd 64 64`
- `ceph osd crush rule ls`: list ra các crush rule
- `ceph osd crush tree --show-shadow`: show cây crush tree
- `ceph osd crush rule create-replicated hdd-rule default host hdd`: tạo crush rule . Cú pháp tổng quát `ceph osd crush rule create-replicated <NAME> <ROOT> <FAILURE_DOMAIN> <CLASS>`
  - Trong đó: hdd-rule:  tên của CRUSH rule (name) – tức là label mà sau này bạn dùng khi tạo/gán pool: --crush-rule hdd-rule, sau này sẽ gán cho pool (ví dụ khi tạo pool: --crush-rule hdd-rule). default: CRUSH root, thường là bucket root mặc định chứa toàn bộ host/OSD (root default trong crush map). 
  - Giải thích từng tham số (theo cú pháp ceph osd crush rule create-replicated NAME ROOT FAILURE_DOMAIN CLASS).
    - hdd-rule (NAME): Tên của CRUSH rule, sau này gán cho pool (ví dụ khi ceph osd pool create ... crush_rule=hdd-rule).
​    - default (ROOT): Root của cây CRUSH, thường là bucket “default” chứa toàn bộ host/OSD trong cluster.
​    - host (FAILURE_DOMAIN): Failure domain; mỗi replica sẽ được đặt trên host khác nhau để tránh mất dữ liệu nếu một node chết.
​    - hdd (CLASS): Device class mục tiêu; rule này chỉ chọn các OSD được gắn class hdd (không dùng SSD/NVMe).
​

Ý nghĩa thực tế: sau khi tạo rule này và gán cho một pool, mọi PG/replica của pool đó sẽ nằm trên các OSD HDD, trải đều trên nhiều host khác nhau, phù hợp làm pool “cold/slow storage” tách riêng với pool SSD.

Trong Ceph, failure domain là một “vùng lỗi” – tức một nhóm tài nguyên mà nếu hỏng thì có thể mất/mất kết nối toàn bộ nhóm đó cùng lúc (ví dụ 1 disk, 1 host, 1 rack, 1 room,…). Ceph dùng failure domain để đảm bảo các bản replica/erasure-chunk của cùng một dữ liệu không nằm trong cùng một vùng lỗi. Ví dụ chọn failure domain là host tức là mỗi replica sẽ nằm trên host vật lý khác nhau; nếu chọn rack thì các replica sẽ nằm ở nhiều rack khác nhau để chịu được sự cố mất cả một rack. Trong CRUSH map, hierarchy có các bucket như osd → host → rack → room → datacenter, và rule sẽ nói “hãy đặt replica trải trên failure domain X”. Khi thiết kế, chọn failure domain càng lớn (rack/room/DC) thì độ bền dữ liệu càng cao nhưng cần nhiều tài nguyên hơn để phân tán
- `ceph osd pool set pool-hdd crush_rule hdd-rule`: gán pool vào crush rule
- `ceph osd pool get pool-hdd crush_rule` : kiểm tra xem pool gắn với crush rule nào

<img width="808" height="493" alt="image" src="https://github.com/user-attachments/assets/87cde9c2-5372-41f4-936f-5f739ebc4141" />

Kết quả dữ liệu luôn được lưu vào ổ hdd trên 3 node khác nhau (khác với ví dụ trước là dữ liệu có lưu ở ssd) nếu dùng pool pool-hdd

---

Cách cung cấp ceph cho cụm promox
- B1: tạo pool
- B2: tạo user có quyền sử dụng pool (bước này đồng thời sinh ra key)
- B3: copy key vào máy promox cần dùng ceph
- B4: Trong trang quản lý của promox tạo storage dạng RBD trỏ đến pool ở trên, monitor(s) là list các ceph nodes có cài mon
