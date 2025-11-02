Dùng case và shift để xử lý parameter câu lệnh

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
