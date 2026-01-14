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
##### 16. List ra các systemd service đang chạy
`systemctl list-units --type=service --state=running`
