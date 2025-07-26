# 1. Tạo heading

Dùng ký tự "#" để tạo heading, càng nhiều dầu "#" chữ càng nhỏ

# heading 1
## heading 2
### heading 3
#### heading 4
##### heading 5
###### heading 6

---

# 2. Tạo mục con

Có thể tạo mục con bằng dấu "-", khoảng trắng hoặc số

- first item
- second item
  - indent
     1. inner
   
1. Mục đầu tiên
2. Mục thứ hai
3. Mục thứ ba

---

## 3. Tạo menu liên kết đến các phần trong bài viết

1. [Menu 1](#custom_name_1)

2. [Menu 2](#custom_name_2)

    1. [Menu 2.1](#sub_1)

    2. [Menu 2.2](#sub_2)

3. [Menu 3](#custom_name_3)


#### <a name="custom_name_1"></a> 1. Tiêu đề 1
#### <a name="custom_name_2"></a> 2. Tiêu đề 2
##### <a name="sub_1"></a> 2.1 Tiêu đề 2.1
##### <a name="sub_2"></a> 2.2 Tiêu đề 2.2
#### <a name="custom_name_3"></a> 3. Tiêu đề 3

---

## 4. Tạo gạch dài để phân trang

Dùng 3 dấu "---"

---

## 5. Tạo quote

> Any quote

---

# 6. Tạo ký tự đặc biệt

Dùng 2 dấu "`" để bôi nền 1 chữ  : Bold something `hello`

Dùng 2 dấu "**" để in đậm, nghiêng và kẻ ngang: How to bold a character **Bold** and italic *Italic* and delete ~~delete~~

---

## 7. Tạo code block cho từng ngôn ngữ code

- Block không xác định ngôn ngữ
```
function test() {
  console.log("notice the blank line before this function?");
}
```

- Block cho ruby langugue
  
```ruby
require 'redcarpet'
markdown = Redcarpet.new("Hello World!")
puts markdown.to_html
```

- Block cho bash
```bash
echo 'hello world'
```

- Block cho javascript

```javascript
let num = Math.random();
```
---

## 8. Tạo bảng
Để tạo bảng cần dùng tối thiểu 3 dấu "-"

Dùng dấu ":" để căn lề trái hoặc phải

VD1:

| Rank | Languages |
|-----:|-----------|
|     1| JavaScript|
|     2| Python    |
|     3| SQL       |

VD2:

| Rank | Languages | Random |
| --- | --- | --- |
| 1 | Java | abc |
| 2 | Python | abc |

---

## 9. Tạo Collapse để ẩn thông tin

<details>
<summary>My top languages</summary>

| Rank | Languages |
|-----:|-----------|
|     1| JavaScript|
|     2| Python    |
|     3| SQL       |

</details>

---

## 10. Tạo comment (không hiển thị khi xem, chỉ hiển thị trong code)

<!-- TO DO: add more details about me later -->

---

# 11. Tạo liên kết URL
        
[Put link here](https://google.com)

---

## 12. Chèn ảnh
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://user-images.githubusercontent.com/25423296/163456776-7f95b81a-f1ed-45f7-b7ab-8fa810d529fa.png">
  <source media="(prefers-color-scheme: light)" srcset="https://user-images.githubusercontent.com/25423296/163456779-a8556205-d0a5-45e2-ac17-42d089e3c3f8.png">
  <img alt="Shows an illustrated sun in light mode and a moon with stars in dark mode." src="https://user-images.githubusercontent.com/25423296/163456779-a8556205-d0a5-45e2-ac17-42d089e3c3f8.png">
</picture>

![alt text](http://picsum.photos/200/200)

---
