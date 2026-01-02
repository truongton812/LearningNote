# CEPH

## 1. Giới thiệu

Ceph là distributed storage mã nguồn mở, cho phép lưu trữ mọi dạng dữ liệu object (như ảnh/video), block (dữ liệu thô cho DB/VM), file (POSIX như thư mục). Ceph được thiết kế để mở rộng theo chiều ngang, chịu lỗi cao và loại bỏ điểm lỗi đơn

### 1.1. Cách Ceph hoạt động
- Tất cả dữ liệu đều được phân mảnh thành object nhỏ (mặc định là 4Mb) trong RADOS pool, sau đó Ceph tự động replicate và tự động phân bổ dữ liệu trên các OSD bằng thuật toán CRUSH. Người dùng chỉ cần chọn interface đúng để sử dụng: RGW cho object, RBD cho block, CephFS cho file​
- Ví dụ thực tế:
  - Lưu file log: Mount CephFS như NFS.
  - Lưu DB data: Attach RBD như EBS volume trên k8s.
  - Lưu backup S3: Upload qua RGW API.

### 1.2 Kiến trúc 1 cụm Ceph
#### 1.2.1. Các thành phần chính của Ceph:
Ceph bao gồm các daemon hoạt động phân tán trên nhiều node:
- Monitors (MON): MON duy trì cluster map (bản đồ trạng thái cụm) chứa thông tin về trạng thái cụm. Trong 1 cụm Ceph thường có 3-5 MON để HA cho nhau. Các MON sử dụng thuật toán đồng thuận Paxos để nhất quán cluster map. 
Cluster map được lưu dưới dạng key-value trong DB store (như RocksDB). Client và các daemon khác truy vấn cluster map để định vị dữ liệu
- Object Storage Daemon (OSD): OSD ánh xạ đến disk để lưu trữ dữ liệu(1 OSD = 1 disk), xử lý read/write, replication/recovery, heartbeat với MON/MGR. Số lượng OSD quyết định capacity/performance, dùng erasure coding tiết kiệm dung lượng.
- Ceph Manager (MGR): MGR thu thập metrics, cung cấp dashboard web, REST API, orchestrator (cephadm) tự động deploy daemon. MGR giám sát và vận hành cluster (MON chỉ lưu giữ thông tin, còn MGR mới là thành phần thực thi). Ít nhất 2 MGR cho HA, tích hợp Prometheus exporter. Khi tương tác với dashboard web đồng nghĩa với tương tác với MGR
​- Metadata Server (MDS): MDS chỉ cần cho CephFS (file system), quản lý metadata (inode, directory, permission), cache để tránh tải OSD. Không dùng CephFS (object/block) thì không cần deploy MDS.
- RADOS Gateway (RGW): RGW cung cấp giao diện object storage tương thích S3/Swift, cho phép truy cập RESTful vào RADOS.

#### 1.2.2. Kiến trúc phân lớp
##### Ceph cluster có 2 plane là control plane và data plane
- Control plane bao gồm Ceph Monitors (MON) và Managers (MGR), chịu trách nhiệm duy trì cluster map (gồm mon map, OSD map, PG map, MDS map, CRUSH map). MONs không xử lý dữ liệu mà chỉ quản lý trạng thái toàn cục
- Data plane chủ yếu là OSD Daemons, xử lý lưu trữ objects thực tế trên đĩa, replication/erasure coding, rebalancing và phục hồi dữ liệu theo CRUSH algorithm mà client sử dụng trực tiếp để định vị PGs (placement groups). Client tương tác thẳng với primary OSD mà không qua gateway trung tâm, tăng hiệu suất. MDS/RGW giúp mở rộng data plane cho file/object services
​
Luồng hoạt động: Client lấy cluster map từ control plane (MON), sau đó dùng CRUSH tính toán vị trí dữ liệu trên data plane (OSD). 

##### Cách Ceph phân bổ daemon

<img width="1520" height="881" alt="image" src="https://github.com/user-attachments/assets/0a3091dd-bf43-49d1-a800-50298c18e1f8" />

Trong cụm Ceph các daemons được phân bổ để đảm bảo high availability (HA), tránh single point of failure và tuân thủ quy tắc quorum cho MON/MGR, với OSD chạy trên tất cả node để cân bằng tải dữ liệu. 

Ví dụ cluster Ceph với 5 nodes:
- Triển khai 3 hoặc 5 MON trên các node để đạt quorum
- 2-3 MGR để đảm bảo cho redundancy, vì MGR hỗ trợ dashboard và orchestration.
- OSD được phân bổ đều trên tất cả các node, mỗi node chạy nhiều OSD tương ứng số disk


### 1.3. Các lệnh làm việc với Ceph
- `ceph orch ls` : liệt kê ra tất cả daemon trên các host (output sẽ giống xem trong Administration > Services trên dashboard)
- `ceph orch ls mon` : liệt kê ra các mon
- `ceph orch ps` : liệt kê ra các container (alertmanager, ceph-exporter, crash - lưu log thì phải, grafana, mgr, mon, node-exporter, prometheus,..) (hay còn gọi là daemon) trên tất cả các node. Thêm option --refresh để lấy mới nhất
- `ceph orch ps` : liệt kê tất cả daemon trên 1 host cụ thể
- `ceph orch apply mon --unmanaged` : không muốn tự động nữa ?? Do khi khởi tạo thì 1 node add vào cluster sẽ có đủ các chức năng, dùng lệnh này để ta tự add chức năng thủ công cho node
- `ceph orch apply mon "3 <hostname1> <hostname2> <hostname3>"`: triển khai mon lên 3 node (??)
- `ceph orch daemon stop/start rgw.myrgw.node3.abcxyz`: stop/start daemon
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

```
 pool là “vùng lưu trữ logic” dùng để nhóm và quản lý các object, giống như một namespace hoặc project riêng trong cluster.
​

Pool là gì
- Mỗi pool là một tập hợp logic các object; khi client ghi dữ liệu, object luôn gắn với một pool cụ thể (ví dụ rbd, cephfs_data, cephfs_metadata).
​- Một pool được chia nhỏ thành nhiều Placement Group (PG); mỗi PG lại được CRUSH map lên các OSD, nên dữ liệu của một pool vẫn được phân tán trên toàn cluster.
​
Các thuộc tính quan trọng của pool
- Kiểu bảo vệ dữ liệu: replicated (số replica) hoặc erasure-coded; giá trị này quyết định cách Ceph nhân bản/chia nhỏ dữ liệu.
- Số lượng PG: tham số pg_num, pgp_num ảnh hưởng tới mức độ phân tán và cân bằng load của pool; mỗi PG chỉ thuộc duy nhất một pool.
- CRUSH rule: mỗi pool gắn với một CRUSH rule để xác định dữ liệu trong pool đặt ở class nào (hdd/ssd) và failure domain (osd/host/rack).​

Tác dụng thực tế
- Tách workload: ví dụ một pool HDD cho backup, một pool SSD cho VM/RBD, một pool khác cho CephFS metadata.
- Kiểm soát policy: có thể set quota, compression, replication size, min_size… theo từng pool, thay vì áp dụng toàn cluster
```
- Khi tạo cluster mặc định có pool .mgr
- `ceph osd pool ls`: liệt kê các pool
- `ceph osd crush class ls`: liệt kê loại disk là hdd hay sdd
- `ceph osd pool create test-pool 32 32`: tạo pool tên test-pool (tìm hiểu 32 là gì - lquan đến pg)
- `ceph osd pool set test-pool size 3`: dữ liệu ghi vào sẽ ghi ở 3 nơi (3 copy).
- `ceph osd pool set test-pool min_size 2`: y/c ceph trong trường hợp lỗi thì phải giữ tối thiểu là 2
- `for pool in $(ceph osd pool ls); do echo "Pool: $pool" && ceph osd pool get $pool size & ceph osd pool get $pool pg_num && echo "----"; done`: script để in ra thông tin size và pg_num của pool
- `for pool in $(ceph osd pool ls | grep rgw); do ceph osd pool application enable $pool rgw; done`: script để enable ứng dụng "rgw" cho các rgw pool, lệnh giúp cấu hình các pool dành cho object storage (RGW) để Ceph nhận diện và quản lý đúng cách, hỗ trợ tính năng như quota, placement rules.
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
​- CRUSH rule là “policy” nói cho CRUSH biết cần chọn OSD nào, ở failure domain nào, bao nhiêu replica/chunk

- Ví dụ dùng thực tế:
  - Bạn có thể tạo một rule cho pool replicated 3 bản sao, yêu cầu mỗi bản nằm trên host khác nhau và chỉ dùng class ssd: ceph osd crush rule create-replicated fast default host ssd.
  - Hoặc rule cho pool erasure-coded, chọn OSD trải đều trên nhiều rack, dùng chế độ indep để xử lý khi OSD bị down.
  
- Áp dụng cho pool:
  - Mỗi pool Ceph sẽ gán với một CRUSH rule; rule này quyết định placement và chiến lược replication/erasure coding của toàn bộ PG trong pool đó.
  - Thay đổi rule (hoặc gán rule khác cho pool) sẽ làm Ceph remap lại PG để đáp ứng chính sách mới, ví dụ dàn lại dữ liệu sang SSD hoặc sang DC khác

Cách thức sử dụng CRUSH rule:
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

Tìm hiểu giao thức RBD

Ngoài RBD, Ceph còn có ba “cổng” chính để client kết nối:

1. CephFS (Ceph File System)
Cung cấp giao thức filesystem POSIX (mount như NFS/EXT4) qua kernel client hoặc FUSE.

Phù hợp chia sẻ file/directory cho app, web server, Hadoop, container…

2. RADOS Gateway (RGW – Object Storage)
Expose Ceph như object storage với REST API tương thích Amazon S3 và OpenStack Swift.

Dùng cho app cần S3/Swift API (microservice, backup, MinIO-like use case).

3. Native librados / libcephfs APIs
Ứng dụng có thể gọi librados (C/C++/Java/Python/…​) để truy cập trực tiếp layer RADOS, bỏ qua RBD/RGW/FS.

Dùng khi cần tối ưu performance, custom protocol, hoặc build service tầng trên (VD: tự viết object service).

---

Ceph object gateway - Rados gateway (RGW)

Các bước:
- Add label rgw cho host muốn triển khai rgw trong cụm
- Triển khai rgw lên 2 host: `ceph orch apply rgw myrgw --placement="2 <hostname1> <hostname2>" --port=7480` 
- Kiểm tra lại bằng `ceph orch ps | grep rgw` và `ceph orch ls rgw`
- `radosgw-admin zone get`
- `ceph osd pool ls | grep rgw`
- Tạo file cấu hình
<img width="681" height="209" alt="image" src="https://github.com/user-attachments/assets/73e25b26-9692-4550-855e-ca3a4928f3ed" />
- `radosgw-admin user create...`: tạo user -> lấy được accesss key và secret key
- `radosgw-admin user list`: list ra các user
- `radosgw-admin user info --uid=<name>`: lấy thông tin user
- `radosgw-admin quota set --uid=<..> --quota-scope=user --max-size=10G` : set quota cho user
- `radosgw-admin quota enable --uid=<..> --quota-scope=user` : enable quota
- Cấu hình SSL
<img width="552" height="205" alt="image" src="https://github.com/user-attachments/assets/843fe622-6b99-4cb7-8fd6-f2954aceae18" />
-> ceph orch apply -i /root/rgw-ssl-spec.yaml
  
<img width="1520" height="1439" alt="image" src="https://github.com/user-attachments/assets/27a4ab3e-48c0-4f3a-bf2c-2a4a40b59e68" />

---
Để lưu trữ dữ liệu database trên Kubernetes (k8s), sử dụng Ceph Block storage (RBD - RADOS Block Device) là lựa chọn tối ưu nhất vì nó cung cấp hiệu suất cao, độ trễ thấp và truy cập raw block phù hợp với workload transactional của database như PostgreSQL, MySQL hoặc MongoDB.
​
Triển khai trên k8s

Sử dụng Rook operator để deploy Ceph cluster trên k8s, sau đó tạo StorageClass cho RBD provisioner (ví dụ: rook-ceph-block). PVC sẽ map RBD image làm block device, mount vào pod database với volumeMode: Block hoặc Filesystem. Điều này tận dụng tự động scale và replication của Ceph, phù hợp với môi trường DevOps AWS/K8s của bạn


So sánh các loại Ceph storage

| Loại storage | Đặc điểm chính | Phù hợp cho database? | Lý do |
|--------------|----------------|-----------------------|-------|
| Block (RBD) | Truy cập raw block device qua CSI driver (như Rook-Ceph), ReadWriteOnce, high IOPS/low latency | Có, khuyến nghị | Database cần hiệu suất cao, ghi ngẫu nhiên nhanh; tích hợp trực tiếp với k8s PV/PVC.[web:11][web:12][web:16] |
| File (CephFS) | POSIX filesystem, hỗ trợ ReadWriteMany, chia sẻ multi-pod | Không lý tưởng | Tốt cho shared files/home directories, nhưng overhead cao hơn cho DB workload.[web:13][web:16] |
| Object (RGW/S3) | Object-based qua API (S3/Swift), không mount như filesystem | Không | Dùng cho backups/blobs, không hỗ trợ modify thường xuyên như DB.[web:14][web:16] |


--

Object

Trong Ceph, object là đơn vị dữ liệu nhỏ nhất mà cluster lưu trữ và quản lý, giống “file” nhưng ở mức hệ thống phân tán.
​

Đặc điểm của object
- Mỗi object có: dữ liệu (payload), tên object (object id) và metadata đi kèm.
- Object không nằm trực tiếp trên OSD riêng lẻ mà thuộc về một Placement Group (PG), rồi PG đó được CRUSH map tới nhiều OSD khác nhau.
​

Quan hệ với pool, RBD, CephFS, RGW
- Khi bạn ghi dữ liệu vào pool (qua RBD, CephFS hoặc S3/RGW), Ceph sẽ chia dữ liệu thành nhiều object.
- Ví dụ một image RBD 10 GB có thể gồm hàng nghìn object kích thước 4 MB; Ceph quản lý replication/erasure coding trên từng object này giữa các OSD.

