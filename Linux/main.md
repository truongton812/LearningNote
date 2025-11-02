### Các lệnh linux thông dụng

##### xem thông tin 1 directory mà không show file bên trong, dùng:

```bash
ls -ld <tên_directory>
```
##### ```sudo -i -u postgres```

-i (viết tắt của "interactive login shell"): Đây là một option trong sudo dùng để mô phỏng phiên đăng nhập tương tác của user được chuyển sang (ở đây là user postgres). Nó sẽ thực thi môi trường shell như khi user đó đăng nhập trực tiếp, bao gồm thiết lập biến môi trường như HOME, tải các file cấu hình shell của user đó, v.v. Điều này giúp tạo ra môi trường làm việc giống phiên đăng nhập chính thức của user postgres.

-u postgres: Đây là option trong sudo để chỉ định user mà lệnh sẽ chạy dưới quyền của user đó. Ở đây, lệnh sẽ chạy với quyền của user postgres, thường là user quản lý cơ sở dữ liệu PostgreSQL.

Kết hợp lại, sudo -i -u postgres nghĩa là "chạy một phiên shell tương tác như user postgres", cho phép thực thi các lệnh tiếp theo trong môi trường của user postgres với đầy đủ quyền và biến môi trường của user đó mà không cần đăng nhập trực tiếp bằng tài khoản postgres. Khi dùng sudo -i -u postgres, sẽ mở một phiên shell giống hệt như user postgres đăng nhập trực tiếp, tức là tái tạo biến môi trường, thư mục home, và tải các file cấu hình shell của user postgres. Điều này quan trọng khi thực thi các lệnh cần môi trường đầy đủ của user postgres, ví dụ các lệnh PostgreSQL mà user đó thường dùng. Nếu không có -i, có thể một số lệnh bị lỗi hoặc không đúng do môi trường thiếu biến hoặc cấu hình

##### Command -v

Lệnh `command -v` trong shell dùng để kiểm tra xem một lệnh hoặc chương trình có tồn tại và có thể thực thi được trên hệ thống hay không.  

Nếu lệnh hoặc chương trình không tồn tại, command -v không trả về gì và có trạng thái thoát không thành công.

Ví dụ, command -v ls sẽ trả về đường dẫn như /bin/ls nếu ls có trên hệ thống.



### Thêm ổ vào server

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
