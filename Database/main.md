## Các lệnh làm việc với database

Restore data using mysql command
mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" < "$BACKUP_FILE"
Nếu đang trong db thì dùng lệnh source <path>;

để kết nối đến db:
Mysql -h 192.168.1.100 -P 3306 -u shoeshop -p

1.
Create database shoeshop; #Tạo database

2.
CREATE USER 'shoeshop'@'%' IDENTIFIED BY 'shoeshop';
Giải thích:
CREATE USER: Câu lệnh tạo tài khoản người dùng mới trong MariaDB.
'shoeshop'@'%': Tên người dùng là shoeshop.
Phần '%' là hostname, nghĩa là người dùng shoeshop có thể kết nối từ bất kỳ địa chỉ IP hoặc máy chủ nào (ký tự % là wildcard đại diện cho tất cả host).
Trong câu lệnh MariaDB như 'shoeshop'@'%', ký hiệu @ dùng để phân tách tên người dùng và host (máy chủ) mà từ đó người dùng đó có thể kết nối.
IDENTIFIED BY 'shoeshop': Thiết lập mật khẩu cho người dùng là shoeshop.


Sau khi tạo user, bạn thường phải cấp quyền cho user đó (như truy cập cơ sở dữ liệu cụ thể) bằng lệnh GRANT.
GRANT ALL PRIVILEGES ON shoeshop.* TO 'shoeshop'@'%';
FLUSH PRIVILEGES;

Ý nghĩa từng phần:
GRANT ALL PRIVILEGES

Cấp tất cả các quyền (SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, v.v.) cho user.

Tương đương user này có thể làm mọi thao tác trên phạm vi mà ta chỉ định.

ON shopdb.*

shopdb là tên cơ sở dữ liệu.

Dấu * nghĩa là áp dụng cho tất cả các bảng và đối tượng bên trong database shopdb.

Nghĩa là user được cấp quyền trên toàn bộ database shopdb.

TO 'shoeshop'@'%'

'shoeshop' là tên người dùng trong MySQL.

@'%' nghĩa là user này có thể kết nối từ bất kỳ địa chỉ IP nào (mọi host).

Nếu bạn muốn giới hạn truy cập chỉ từ một IP hoặc localhost, bạn có thể dùng:

@'localhost' → chỉ cho phép user connect trên máy cài MySQL.

@'192.168.1.%' → chỉ cho phép từ dải IP nội bộ 192.168.1.x.


Lưu ý bảo mật:
ALL PRIVILEGES + @'%' khá nguy hiểm trong môi trường production vì user này có toàn quyền và có thể đăng nhập từ mọi nơi.

Thông thường, để an toàn hơn, chỉ nên:

Cấp đúng quyền cần thiết (SELECT, INSERT, UPDATE… thay vì ALL PRIVILEGES).

Giới hạn host (@'localhost' hoặc IP cụ thể thay vì %).

<img width="627" height="388" alt="image" src="https://github.com/user-attachments/assets/884c869d-d3db-4d39-b273-4c8b8485c9a6" />



3.   
Để truy vấn danh sách các user hiện có trong MariaDB, bạn có thể dùng câu lệnh SQL sau:
sql
SELECT User, Host FROM mysql.user;

User: Tên người dùng (user) trong MariaDB.
Host: Máy chủ (host) mà user đó được phép đăng nhập từ đó.
Câu lệnh này sẽ trả về danh sách các user cùng với host liên kết, giúp bạn biết được các tài khoản được tạo trên hệ thống.

4.
các lệnh để kiểm tra db
Show databases;
Use shoeshop;
Show tables;

---

Create a Database:
You can create a database using the mysqladmin command. Replace dbname with the desired name for your database.

mysqladmin -u root -p create dbname
You will be prompted to enter the password for the root user.

Create a User:
You can create a user using the mysql command. Replace username and password with the desired username and password for the new user.


mysql -u root -p -e "CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';"
Grant Privileges:
You can grant privileges to the user on the database using the mysql command. Replace dbname with the name of the database and username with the username you created earlier.

mysql -u root -p -e "GRANT ALL PRIVILEGES ON dbname.* TO 'username'@'localhost';"
Flush Privileges:
After granting privileges, run the following command to apply the changes:


mysql -u root -p -e "FLUSH PRIVILEGES;"
This ensures that the changes are applied immediately.
