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
