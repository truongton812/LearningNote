### Các lệnh linux thông dụng

xem thông tin 1 directory mà không show file bên trong, dùng:

```bash
ls -ld <tên_directory>
```

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
