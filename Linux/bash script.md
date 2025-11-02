#### Dùng case và shift để xử lý parameter câu lệnh

```
#!/bin/bash
while [[ $# -gt 0 ]]; do
  case "$1" in
    -n)
      echo "Name is $2"
      shift 2
      ;;
    -a)
      echo "Age is $2"
      shift 2
      ;;
    -g)
      echo "Gender is $2"
      shift 2
      ;;
    *)
      echo "Usage: $0 -n <name> -a <age> -g <gender>"
      exit 1
      ;;
  esac
done

```

Giải thích:
- Mỗi lần shift, các argument sẽ dịch lên một vị trí, còn $# sẽ giảm đi 1 hoặc 2
- khi không dùng vòng lặp thì câu lệnh `case "$1" in ... esac` trong bash chỉ xử lý đúng một lần với giá trị hiện tại của $1 tại thời điểm script chạy. Nếu bạn muốn xử lý nhiều argument, cần đặt `case "$1"` bên trong một vòng lặp (while, for) để tự động lặp lại kiểm tra và xử lý với từng argument đến khi hết (tức là không còn $1 nữa).
- shift sẽ dịch tất cả các tham số sang trái một vị trí (hoặc nhiều vị trí nếu có tham số số). Ví dụ: $2 sẽ trở thành $1, $3 trở thành $2, và $1 ban đầu bị loại bỏ.
- Lệnh shift sẽ tác động tới toàn bộ script, không chỉ riêng trong case. Sau khi shift, tất cả các biến $1, $2, ... sẽ bị cập nhật lại theo các argument còn lại. Điều này đúng với toàn bộ script, chứ không chỉ trong một block lệnh nào đó.

#### -N
Điều kiện if [[ -n "$2" ]]; then trong shell script dùng để kiểm tra xem tham số vị trí thứ hai ($2) có không rỗng hay không.

Nếu $2 không rỗng thì điều kiện trả về true, và khối lệnh trong then sẽ được thực thi.

Nếu $2 rỗng hoặc không được truyền vào, điều kiện trả về false và phần then sẽ bị bỏ qua.

#### if [[ -z "$env" ]] 

Điều kiện if [[ -z "$env" ]] trong shell script là để kiểm tra xem biến $env có rỗng hay không.

Nếu $env rỗng thì điều kiện trả về true, phần lệnh trong then sẽ được thực thi.

Nếu $env có bất kỳ giá trị nào (không rỗng), điều kiện trả về false, phần then sẽ bị bỏ qua.

#### local custom_dir=$(yq eval '.directories // ""' "${BASE_DIR}/$config_file")

// là toán tử trong yq, có chức năng cung cấp giá trị mặc định khi trường directories không tồn tại hoặc là chuỗi rỗng

 -> dòng này kiểm tra trường directories trong file YAML, lấy ra giá trị nếu có, còn nếu không có thì mặc định là chuỗi rỗng, rồi lưu vào biến custom_dir

#### Đoạn code
```
while IFS= read -r service; do
  services+=("$service")
done <<< $(get_env_services "$env")
```

- get_env_services "$env": gọi hàm hoặc lệnh get_env_services với tham số $env, trả về danh sách các dịch vụ, mỗi dịch vụ trên một dòng.

- <<< $(...): sử dụng here-string để chuyển đầu ra của lệnh get_env_services làm đầu vào của vòng lặp while.

- while IFS= read -r service: vòng lặp đọc từng dòng một từ đầu vào (output của get_env_services), gán dòng đó vào biến service. IFS= để không cắt bỏ khoảng trắng đầu/cuối của dòng. -r để đọc nguyên vẹn dấu \ mà không hiểu là escape ký tự.

- `services+=("$service")`: thêm từng dòng service vào mảng services từng phần tử một.

Tóm lại đoạn code này lấy danh sách các dịch vụ in từng dòng từ get_env_services. Đọc từng dòng ra riêng biệt. Thêm từng dịch vụ vào mảng services.


#### Câu lệnh

```
if ! create_dispatch_yaml "$env" maintenance "$dispatch"; then
```

Gọi hàm create_dispatch_yaml với 3 tham số.

Đợi hàm thực thi và kiểm tra giá trị trả về (exit code).

Dấu ! phủ định kết quả exit code, tức là nếu hàm trả về lỗi (mã khác 0), điều kiện trong if sẽ đúng và khối lệnh trong then sẽ được thực thi.

Như vậy, hàm luôn được gọi, nhưng phần then chỉ chạy khi hàm create_dispatch_yaml gặp lỗi trong quá trình thực thi.
