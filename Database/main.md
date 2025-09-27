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

---

# Quản lý User và Phân quyền trong PostgreSQL

Trong PostgreSQL, việc quản lý truy cập được thực hiện thông qua hệ thống "role". Một `USER` về cơ bản là một `ROLE` có thuộc tính `LOGIN`.

## Tạo và Quản lý User (Role)

Các lệnh cơ bản để tạo, sửa, xóa và xem danh sách người dùng.

*   **Tạo user mới:**
    Lệnh này tạo một user mới có quyền đăng nhập và gán mật khẩu đã được mã hóa.

    ```sql
    CREATE USER ten_user WITH PASSWORD 'mat_khau_an_toan';
    ```

*   **Sửa đổi thuộc tính user:**
    Bạn có thể thay đổi các thuộc tính của user, ví dụ như cấp hoặc thu hồi quyền superuser.

    ```sql
    -- Cấp quyền superuser
    ALTER USER ten_user WITH SUPERUSER;

    -- Thu hồi quyền superuser
    ALTER USER ten_user WITH NOSUPERUSER;
    ```

*   **Xóa user:**

    ```sql
    DROP USER ten_user;
    ```

*   **Liệt kê tất cả user và role:**
    Sử dụng lệnh `\du` trong giao diện dòng lệnh `psql`.

    ```bash
    \du
    ```

## Gán Quyền với Lệnh `GRANT`

Lệnh `GRANT` được dùng để cấp các đặc quyền cụ thể trên các đối tượng cơ sở dữ liệu (database, schema, table) cho một user hoặc role.

**Cú pháp chung:**
`GRANT [quyền] ON [loại_đối_tượng] [tên_đối_tượng] TO [tên_user_hoặc_role];`

### Các quyền phổ biến

| Quyền | Mô tả | Đối tượng áp dụng |
| :--- | :--- | :--- |
| **`CONNECT`** | Cho phép user kết nối đến cơ sở dữ liệu. | `DATABASE` |
| **`CREATE`** | Cho phép user tạo đối tượng mới (ví dụ: bảng, schema). | `DATABASE`, `SCHEMA` |
| **`USAGE`** | Cho phép user truy cập các đối tượng trong một schema (cần thiết để thao tác với bảng). | `SCHEMA` |
| **`SELECT`** | Cho phép user đọc dữ liệu từ bảng. | `TABLE`, `VIEW` |
| **`INSERT`** | Cho phép user chèn dữ liệu mới vào bảng. | `TABLE` |
| **`UPDATE`** | Cho phép user cập nhật dữ liệu hiện có trong bảng. | `TABLE` |
| **`DELETE`** | Cho phép user xóa dữ liệu khỏi bảng. | `TABLE` |
| **`ALL PRIVILEGES`** | Gán tất cả các quyền có thể có trên một đối tượng. | `DATABASE`, `TABLE`, `SCHEMA` |

## Các ví dụ phân quyền phổ biến

1.  **Cấp quyền kết nối vào database:**
    Đây là quyền cơ bản nhất để user có thể bắt đầu làm việc.
    ```sql
    GRANT CONNECT ON DATABASE ten_database TO ten_user;
    ```
2.  **Cấp quyền sử dụng schema:**
    Sau khi kết nối, user cần quyền `USAGE` trên schema (thường là `public`) để "thấy" các bảng bên trong.
    ```sql
    GRANT USAGE ON SCHEMA public TO ten_user;
    ```
3.  **Cấp quyền thao tác trên tất cả các bảng hiện có:**
    Cho phép user thực hiện các thao tác CRUD (Create, Read, Update, Delete) trên tất cả các bảng đã tồn tại trong schema `public`.
    ```sql
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO ten_user;
    ```

## Quản lý Quyền Nâng cao

### 1. Phân quyền cho các đối tượng sẽ được tạo trong tương lai

Lệnh `GRANT` chỉ áp dụng cho các đối tượng đã tồn tại. Để tự động gán quyền cho các bảng được tạo trong tương lai, hãy sử dụng `ALTER DEFAULT PRIVILEGES`.

Lệnh này đảm bảo rằng `ten_user` sẽ tự động có quyền `SELECT` và `INSERT` trên tất cả các bảng *mới* được tạo trong schema `public` bởi `user_tao_bang`.

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE user_tao_bang IN SCHEMA public
GRANT SELECT, INSERT ON TABLES TO ten_user;
```


### Sử dụng Role như một Nhóm quyền

Để quản lý quyền dễ dàng hơn, bạn có thể tạo một role chung (nhóm quyền), gán các quyền cho role đó, rồi gán role đó cho các user.

**Bước 1: Tạo một role để làm nhóm**

Đầu tiên, tạo một role mới không có thuộc tính `LOGIN`. Role này sẽ hoạt động như một container chứa các quyền.

```
CREATE ROLE read_only_group;
```


**Bước 2: Gán các quyền cần thiết cho nhóm**

Tiếp theo, cấp tất cả các quyền cần thiết (ví dụ: `CONNECT`, `USAGE`, `SELECT`) cho role nhóm vừa tạo.

```
GRANT CONNECT ON DATABASE ten_database TO read_only_group;
GRANT USAGE ON SCHEMA public TO read_only_group;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO read_only_group;
```


**Bước 3: Gán nhóm quyền này cho một user cụ thể**

Cuối cùng, gán role nhóm (`read_only_group`) cho user cuối (`ten_user`). User này sẽ kế thừa tất cả các quyền đã được cấp cho nhóm.

```
GRANT read_only_group TO ten_user;
```

Với cách này, việc quản lý trở nên đơn giản hơn vì bạn chỉ cần thêm hoặc bớt quyền từ `read_only_group` và mọi thành viên của nhóm sẽ tự động được cập nhật.
