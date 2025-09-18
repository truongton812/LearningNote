Thêm ổ vào server

Để thêm ổ sdb vào LVM hiện tại trên ổ sda (Volume Group ubuntu-vg), bạn có thể thực hiện theo các bước dưới đây:

Các bước thêm sdb vào LVM hiện tại
Kiểm tra ổ sdb hiện tại
Kiểm tra ổ sdb và phân vùng nếu có:

bash
lsblk /dev/sdb
Tạo phân vùng kiểu LVM trên sdb (nếu chưa có phân vùng)
Nếu sdb chưa có phân vùng hoặc muốn dùng toàn bộ ổ làm PV, bạn có thể tạo phân vùng mới với mã kiểu LVM (8e) hoặc dùng nguyên ổ /dev/sdb để làm PV trực tiếp (không phân vùng). Cách an toàn là tạo phân vùng:

bash
fdisk /dev/sdb
Tạo phân vùng mới (n)

Đặt kiểu phân vùng là 8e (Linux LVM) bằng lệnh t

Lưu (w)

Tạo Physical Volume (PV) trên phân vùng mới hoặc ổ cứng
Ví dụ phân vùng mới là /dev/sdb1:

bash
pvcreate /dev/sdb1
Thêm PV mới vào Volume Group hiện tại
Thêm /dev/sdb1 vào VG ubuntu-vg:

bash
vgextend ubuntu-vg /dev/sdb1
Kiểm tra lại VG đã mở rộng

bash
vgdisplay ubuntu-vg
Mở rộng Logical Volume hoặc tạo LV mới
Ví dụ mở rộng LV ubuntu-lv để sử dụng thêm không gian mới:

bash
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
Hoặc tạo LV mới trong VG nếu muốn.

Mở rộng filesystem theo LV đã mở rộng
Nếu filesystem là ext4 thì dùng:

bash
resize2fs /dev/ubuntu-vg/ubuntu-lv
Với xfs filesystem sẽ dùng:

bash
xfs_growfs /
Tóm tắt lệnh mẫu
bash
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
Bạn nên backup dữ liệu quan trọng trước khi thực hiện thao tác với ổ đĩa và LVM để tránh rủi ro mất dữ liệu. Nếu cần hướng dẫn chi tiết hoặc gặp lỗi trong quá trình thực hiện, có thể hỏi thêm nhé.

chỉ có 2 option là p và e thôi, không có t

chỉ có 2 option là p và e thôi, không có t
Trường hợp dùng fdisk mà chỉ có 2 option là p (print) và e (extended) mà không thấy tùy chọn t (đổi loại phân vùng) thì khả năng là bạn đang thao tác trên bảng phân vùng kiểu MBR (msdos) và có thể chưa vào đúng chế độ chỉnh sửa phân vùng, hoặc do phiên bản fdisk hơi khác biệt.

Dưới đây là một số cách xử lý để thiết lập loại phân vùng LVM (8e):

Cách kiểm tra và thao tác với fdisk chuẩn
Mở fdisk:

bash
fdisk /dev/sdb
Các bước tạo phân vùng mới:

Gõ n để tạo phân vùng mới (logical hoặc primary)

Chọn số phân vùng, định dạng kích thước

Sau khi tạo phân vùng, gõ p để xem phân vùng hiện tại

Gõ t để đổi loại phân vùng (Change partition type)

Nếu nó không có, có thể gõ m để xem list lệnh hỗ trợ trong fdisk

Nếu t không hiển thị, hoặc không được hỗ trợ:

Giải pháp thay thế
Dùng parted thay vì fdisk để tạo phân vùng với kiểu LVM dễ dàng hơn:

bash
parted /dev/sdb
Trong parted bạn có thể:

mklabel gpt (tạo bảng phân vùng GPT, tốt hơn MBR, nếu có thể)

mkpart primary 0% 100% (tạo phân vùng toàn bộ ổ)

set 1 lvm on (bật flag LVM cho phân vùng 1)

quit

Hoặc nếu không muốn phân vùng, có thể dùng trực tiếp ổ /dev/sdb làm PV luôn, không cần phân vùng:

bash
pvcreate /dev/sdb
Rồi thêm vào VG như bình thường.
