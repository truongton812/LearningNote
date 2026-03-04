### Các lệnh linux thông dụng

##### 1. xem thông tin 1 directory mà không show file bên trong, dùng:

```bash
ls -ld <tên_directory>
```
##### 2. ```sudo -i -u postgres```

-i (viết tắt của "interactive login shell"): Đây là một option trong sudo dùng để mô phỏng phiên đăng nhập tương tác của user được chuyển sang (ở đây là user postgres). Nó sẽ thực thi môi trường shell như khi user đó đăng nhập trực tiếp, bao gồm thiết lập biến môi trường như HOME, tải các file cấu hình shell của user đó, v.v. Điều này giúp tạo ra môi trường làm việc giống phiên đăng nhập chính thức của user postgres.

-u postgres: Đây là option trong sudo để chỉ định user mà lệnh sẽ chạy dưới quyền của user đó. Ở đây, lệnh sẽ chạy với quyền của user postgres, thường là user quản lý cơ sở dữ liệu PostgreSQL.

Kết hợp lại, sudo -i -u postgres nghĩa là "chạy một phiên shell tương tác như user postgres", cho phép thực thi các lệnh tiếp theo trong môi trường của user postgres với đầy đủ quyền và biến môi trường của user đó mà không cần đăng nhập trực tiếp bằng tài khoản postgres. Khi dùng sudo -i -u postgres, sẽ mở một phiên shell giống hệt như user postgres đăng nhập trực tiếp, tức là tái tạo biến môi trường, thư mục home, và tải các file cấu hình shell của user postgres. Điều này quan trọng khi thực thi các lệnh cần môi trường đầy đủ của user postgres, ví dụ các lệnh PostgreSQL mà user đó thường dùng. Nếu không có -i, có thể một số lệnh bị lỗi hoặc không đúng do môi trường thiếu biến hoặc cấu hình

##### 3. Command -v

Lệnh `command -v` trong shell dùng để kiểm tra xem một lệnh hoặc chương trình có tồn tại và có thể thực thi được trên hệ thống hay không.  

Nếu lệnh hoặc chương trình không tồn tại, command -v không trả về gì và có trạng thái thoát không thành công.

Ví dụ, command -v ls sẽ trả về đường dẫn như /bin/ls nếu ls có trên hệ thống.


##### 4. Lọc các tiến trình sử dụng nhiều bộ nhớ

ps aux | awk '{if ($5 != 0 ) print $2,$4,$6,$11}' | sort -k2nr | head -n10

##### 5. Lấy ra những câu lệnh thường xuyên được sử dụng nhất (với Bash Shell)

cat ~/.bash_history | tr "\|\;" "\n" | sed -e "s/^ //g" | cut -d " " -f 1 | sort | uniq -c | sort -nr | head -n 15

##### 6. xóa các dòng trống trong file

sed -i '/^$/d'

##### 7. Đổi tên toàn bộ file trong thư mục để chuyển các space thành underscore

`for i in *; do mv "$i" ${i// /_};done`

##### 8. Cú pháp tắt nhanh một web server đang hoạt động

pgrep -f tcp://[IP]:[PORT] | xargs kill -9

##### 9. Lấy tên của dự án dựa vào github repository

git remote get-url <github_repo_name> | grep -o "\/[a-zA-Z0-9_\-]\+\.git" | sed -E "s/^\/|\.git$//g"


##### 10. Dùng lệnh curl với domain name custom mà không cần sửa dns bằng option resolve

`curl https://secure-ingress.com:31047/service2 --resolve secure-ingress.com:31047:34.105.246.184` -> gọi đến IP 34.105.246.184 bằng domain secure-ingress.com:31047

### Giải thích về storage trong linux
Các khái niệm

1. Ổ đĩa (Disk): Là thiết bị lưu trữ vật lý hoặc ảo (ví dụ: SSD, HDD, ổ đĩa ảo trong cloud). Có thể chia thành nhiều phân vùng và mỗi phân vùng có thể được định dạng, gắn kết (mount) để sử dụng.
​
2. Phân vùng (Partition): Là một vùng logic được chia ra từ một ổ đĩa vật lý hoặc ảo, giúp tổ chức dữ liệu theo mục đích (hệ điều hành, dữ liệu, boot, EFI...). Mỗi phân vùng có thể được định dạng thành hệ thống tập tin (filesystem) riêng biệt và mount vào một thư mục.
​
3. Volume: Volume là một đơn vị lưu trữ logic, có thể là một phân vùng hoặc một tập hợp các phân vùng (như trong LVM). Volume được mount vào một thư mục để sử dụng dữ liệu bên trong nó.
​
4. Thư mục (Directory): Là một cấu trúc logic trong hệ thống tập tin, dùng để tổ chức các file và thư mục con. Thư mục không phải là thiết bị lưu trữ, mà là nơi lưu trữ và quản lý dữ liệu trên hệ thống tập tin đã được mount từ volume hoặc phân vùng.

Lưu ý khi nói đến mount trong Linux, thực chất là mount phân vùng (partition)/volume vào một thư mục, chứ không phải mount trực tiếp cả ổ đĩa

Các lệnh làm việc:

- Lệnh `df -h`: liệt kê ra danh sách các phân vùng/volume/tmpfs ***ĐÃ ĐƯỢC MOUNT** đã được mount (tức đã gắn vào thư mục), bao gồm dung lượng và điểm mount tương ứng.

- Lệnh `lsblk`: Liệt kê ra các ổ đĩa (block device) và các phân vùng của ổ đĩa hiện có trên hệ thống, bao gồm cả những phân vùng chưa được mount. Nếu hệ thống sử dụng LVM (Logical Volume Manager), lsblk cũng sẽ hiển thị các logical volume (volume logic) được tạo ra từ các phân vùng hoặc ổ đĩa.
​
- `fdisk -l`: Tương tự lệnh lsblk nhưng chi tiết hơn

#### Thêm ổ vào server

Để thêm ổ sdb vào LVM hiện tại trên ổ sda (Volume Group ubuntu-vg), bạn có thể thực hiện theo các bước dưới đây:


Tạo phân vùng kiểu LVM trên sdb (nếu chưa có phân vùng)

```
fdisk /dev/sdb # n để tạo phân vùng mới -> p -> t -> 8e -> w
```


Tạo physical volume (PV) từ phân vùng mới , Ví dụ phân vùng mới là /dev/sdb1:

```
pvcreate /dev/sdb1
```

Mở rộng volume group (VG) hiện tại bằng PV mới:

```
vgextend ubuntu-vg /dev/sdb1
```

Kiểm tra lại VG đã mở rộng

```
vgdisplay ubuntu-vg
```

Mở rộng Logical Volume hoặc tạo LV mới. Ví dụ mở rộng LV ubuntu-lv để sử dụng thêm không gian mới:

```
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Mở rộng filesystem theo LV đã mở rộng

Nếu filesystem là ext4 thì dùng:

```
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Với xfs filesystem sẽ dùng:

```
xfs_growfs /
```


Tóm tắt lệnh mẫu
```
# (Tạo phân vùng LVM trên sdb nếu cần)
fdisk /dev/sdb  

# Tạo PV
pvcreate /dev/sdb1

# Mở rộng VG
vgextend ubuntu-vg /dev/sdb1

# Mở rộng LV dùng toàn bộ không gian mới
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Mở rộng filesystem (ext4 hoặc xfs)
resize2fs /dev/ubuntu-vg/ubuntu-lv
# hoặc
xfs_growfs /
```
##### 11. pipe

Lệnh `crictl inspect abcuiauiia | vim - ` gồm phần pipe là ký hiệu | dùng để chuyển đầu ra (output) của lệnh bên trái sang làm đầu vào (input) cho lệnh bên phải.

Dấu - trong lệnh vim - đại diện cho stdin (standard input), nghĩa là Vim sẽ đọc dữ liệu đầu vào từ pipe (|) thay vì từ một file cụ thể trên đĩa.

Đây là tính năng chuẩn của nhiều công cụ Unix/Linux như Vim, cat, grep, kubectl giúp xử lý dữ liệu động một cách linh hoạt

##### 12. Root process

Trên Linux/Unix, một process chạy với UID 0 (root) về nguyên tắc là có “toàn quyền” trên hệ thống (đọc/ghi/xóa hầu hết mọi file, mount/unmount filesystem, thay đổi quyền file và sở hữu (chown/chmod), tạo/sửa user, cấu hình network, dừng dịch vụ, reboot máy, v.v.), nhưng vẫn có một vài ngoại lệ và cơ chế hạn chế bổ sung.

Nếu chỉ nhìn ở mức phân quyền UNIX truyền thống, process chạy dưới root có thể thực hiện mọi lệnh quản trị và phá hỏng hoặc chiếm quyền toàn bộ hệ thống. Tuy nhiên nhiều hệ thống hiện đại dùng thêm cơ chế như Linux capabilities, SELinux, AppArmor, seccomp, namespaces (container) để “chia nhỏ” hoặc giới hạn quyền của root.

Ví dụ: trong container, process là root nhưng chỉ là “root trong namespace đó”; nó vẫn bị giới hạn bởi kernel và policy của host, nên không nhất thiết làm được tất cả mọi thứ trên host.

##### 13. SSH key
Có thể ssh vào 1 máy remote bằng key:
- Tạo private/public key: Dùng ssh-keygen -t rsa trên máy local (client) để sinh cặp khóa.
​- Dán public key vào authorized_keys: Copy nội dung file id_rsa.pub (hoặc tương tự) vào ~/.ssh/authorized_keys trên máy remote (server), đảm bảo quyền 600/644.
​- Dùng private key để truy cập: SSH bằng lệnh ssh -i ~/.ssh/id_rsa user@remote-host hoặc cấu hình tự động trong ~/.ssh/config.
​
Lưu ý bổ sung: Cần kích hoạt PubkeyAuthentication yes trong /etc/ssh/sshd_config trên server và restart SSH service để đảm bảo hoạt động

Lưu ý không thể làm ngược lại (đặt private key ở máy local còn public key ở máy remote) do SSH authentication yêu cầu private key luôn ở client (máy local) và public key ở server (máy remote). Private key phải bí mật, chỉ client giữ. Server chỉ lưu public key để verify, không bao giờ expose private key.

##### 14. Lệnh để xem log ứng dụng systemd
`journalctl --since "2026-01-10 05:30" | grep -E "varnish|systemctl|Stopping varnish"`

##### 15. Các loại pipe
```
Phân biệt ;, &&, ||, | trong command Linux
Hiểu rõ các toán tử giúp viết lệnh shell chính xác, tối ưu hơn:
🔹 ; – Chạy lệnh tuần tự
→ Lệnh thứ hai luôn chạy, bất kể lệnh đầu thành công hay thất bại
Ví dụ:
cmd1 ; cmd2
🔹 && – Chạy khi lệnh trước thành công
→ Lệnh thứ hai chỉ chạy nếu lệnh đầu thành công (exit code = 0)
Ví dụ:
cmd1 && cmd2
🔹 || – Chạy khi lệnh trước thất bại
→ Lệnh thứ hai chỉ chạy nếu lệnh đầu lỗi (exit code ≠ 0)
Ví dụ:
cmd1 || cmd2
🔹 | – Pipe (chuyển output làm input)
→ Kết quả đầu ra của lệnh đầu được truyền làm đầu vào cho lệnh sau
Ví dụ:
cat file.txt | grep "error"
Tóm tắt nhanh:
; → chạy tiếp bất kể thành công hay lỗi
&& → chỉ chạy tiếp khi lệnh trước thành công
|| → chỉ chạy tiếp khi lệnh trước lỗi
| → dùng để nối đầu ra và đầu vào giữa các lệnh
💡 Dùng chính xác các toán tử này sẽ giúp bạn làm chủ terminal hiệu quả hơn!
```

##### 16. Systemd service
Service là gì, cách tạo service cơ bản:
- ví dụ khai báo `ExecStart=/usr/bin/rustfs start`... tìm tài liệu bổ sung thêm

Có 2 cách tạo biến trong khai báo service là dùng Environment= hoặc EnvironmentFile= . EnvironmentFile= trong systemd service dùng để load các biến môi trường từ file text bên ngoài, thay vì hard-code trực tiếp trong file .service. Phương pháp này giúp code sạch sẽ nếu service có nhiều biến và dễ dàng override biến thay vì phải sửa trực tiếp trong .service

Syntax: EnvironmentFile=/path/to/envfile

Format file: Mỗi dòng là VAR=value (không cần quote, comment bắt đầu bằng #)

Ví dụ file /etc/rustfs/env.conf
```
RUSTFS_VOLUMES=/data/rustfs{0..3}
RUSTFS_LOG_LEVEL=info
DB_PASSWORD=secret123
```

Biến từ env file (qua EnvironmentFile=) được load vào environment của process chạy ExecStart, app có thể đọc qua $VAR hoặc os.getenv().

```
[Service]
EnvironmentFile=/etc/rustfs/env.conf
User=rustfs-user
ExecStart=/usr/bin/rustfs-app --log-dir ${RUSTFS_LOG_DIR} --port ${RUSTFS_PORT}
```

Cách xem env của service
`systemctl show rustfs.service -p Environment`

Test process env
`sudo -u rustfs-user env | grep RUSTFS`

Log debug (ExecStartPre)
`ExecStartPre=/bin/sh -c 'env | grep RUSTFS > /tmp/rustfs-env.log'`


##### 16. List ra các systemd service đang chạy
`systemctl list-units --type=service --state=running`

##### 17. Top command
- Load avarage: trung bình tải trong 1 phút, 5 phút và 15 phút gần nhất. Nếu CPU có 1 core mà trung bình tải là 1 nghĩa là đang dùng hết 100% của 1 core đấy. Nếu hệ thống có 4 cores thì max sẽ là 4. Lưu ý Load cao không có nghĩa CPU usage luôn cao, vì có thể do nhiều tiến trình đang chờ I/O (ví dụ: ổ cứng chậm, mạng nghẽn)

- Tasks: 123 total,   1 running, 122 sleeping,   0 stopped,   0 zombie. Trong đó
  - total: tổng số process + thread đang tồn tại trên hệ thống (123).
  - running: số process/thread đang được CPU xử lý ngay lúc này (1).
  - sleeping: số process/thread đang ngủ (chờ I/O, timer, signal, v.v.) (122).
  - stopped: process bị tạm dừng (ví dụ: do SIGSTOP) (0).
  - zombie: process đã chết nhưng chưa được parent thu dọn (0), cách dễ nhất để thu dọn là reboot lại server. Dùng shift + V để xem các child process của parent process
​
Vì sao chỉ có 1 running dù có nhiều app?

Nguyên nhân là do CPU chỉ có thể thực thi 1 process/thread tại 1 thời điểm trên 1 core (trên máy nhiều core thì mỗi core 1 cái). Các ứng dụng khác (trình duyệt, editor, server, v.v.) phần lớn thời gian là sleeping (đang chờ người dùng thao tác, chờ mạng, chờ ổ đĩa, chờ timer). Khi bạn thao tác (gõ phím, click chuột, request mạng), hệ thống sẽ đánh thức process đó → nó chuyển sang trạng thái running trong 1 khoảng thời gian rất ngắn, rồi lại quay về sleeping.

- %Cpu(s):  5.3 us,  2.2 sy,  0.0 ni, 90.9 id,  0.0 wa,  0.0 hi,  1.5 si,  0.0 st. Trong đó
  - us là userspace. Các ứng dụng như nginx, httpd, word, excel,... là chạy ở user space. Thông số ở đây nghĩa là %CPU các ứng dụng ở user space đang chiếm
  - sy là kernal space
  - 0.0 ni là nineness (priority - tìm hiểu thêm). Đại khái là độ ưu tiên CPU dành cho 1 ứng dụng, ta có thể cấu hình
  - id là idle: khoảng tgian CPU idle (ở đây là 90%) - nên chú ý đến chỉ số này. Nếu cao là tốt (ko áp dụng với cloud do sẽ tốn tiền)
  - wa là waiting: khoảng tgian 1 task đang chờ (?) - chỉ số này cao thì xấu

- Ở phần dưới là danh sách các process đang chạy, trường CPU/Mem là resource sử dụng, trường TIME là thời gian thực tế mà ứng dụng sử dụng CPU
- Sắp xếp thông tin:
  - P: Sắp xếp theo %CPU (cao nhất đầu).
  - M: Sắp xếp theo %MEM (bộ nhớ).
  - T: Sắp xếp theo TIME+ (thời gian CPU tích lũy).
  - N: Sắp xếp theo PID (tăng dần).
  - R: Đảo ngược thứ tự sắp xếp.
- Quản Lý Process
  - k: Kill process (nhập PID, rồi signal như 15 cho TERM hoặc 9 cho KILL).
  - c: Toggle hiển thị full command line.
  - u: Filter theo user (nhập tên, Enter để apply, u để clear).
  - n hoặc /: Tìm kiếm PID (nhập số).

##### 18. Grep
- grep -v "pattern" : loại bỏ dòng có <pattern> khỏi kết quả search
- khi ở trong thư mục cũng có thể dùng grep
- Để sắp xếp theo process theo % CPU đang sử dụng thì dùng shift + P
- Để sắp xếp theo process theo % RAM đang sử dụng thì dùng shift + M
- Nhấn K để kill process
- Nhấn D để đổi refresh interval của top

##### 19. du
du -sh * | sort -rh | head -10 -> xem 10 thư mục chiém nhiều lưu lượng nhất

Lưu ý du -sh * và du -sh / khác nhau về phạm vi quét và kết quả hiển thị. 

du -sh * liệt kê kích thước tổng (-s: summary) của tất cả file/thư mục trực tiếp trong thư mục hiện tại (wildcard * của shell mở rộng các item cấp 1).

du -sh / chỉ hiển thị tổng kích thước duy nhất của toàn bộ filesystem root / (bao gồm tất cả thư mục con đệ quy). Muốn xem cụ thể thì dùng du -sh /*

Dùng lệnh sudo du -h --max-depth=1 / | sort -rh cũng tương tự
