#PYTHON coding

Khai báo variable
num1 = 20
num2 = 10

Integer Addition
Addition = num1 + num2
print ("Add of two numbers:", Addition)

Integer Division
Div = num1 // num2
print ("Division of two numberas:", Div)

Remainder (phần dư)
rem = num1 % num2
print ("Remainder:", rem)

Substraction of num1 & num2

result2 = num1 - num2
print ("Substraction:", result2)

Multiplication of num1 & num2

result3 = num1 * num2
print ("Multiplication:", result3)



Division of num1 & num2

result4 = num1 / num2
print ("Division:", result4)

round(3.14159265359, 2)  -> làm tròn 2 chữ số
result2 = num1 % num2 -> lấy phần dư
result3 = abs(-7) -> lấy giá trị tuyệt đối

---
Common function

- len(text) : lấy độ dài string
- text.upper(): đổi thành chuỗi in hoa
- text.lower(): đổi thành chuỗi in thường
- text.replace("string1","string2"): Thay thế chuỗi
- split: tách chuỗi
VD1:
text = "learning,something,new"
new_text = text.split( )
Kết quả: `new_text = ["learning","something","new"]`

VD2:
arn = "arn:partition:service:region:account-id:resource-type/resource-id"
new_arn = arn.split("/")
Kết quả: `new_arn = ["arn:partition:service:region:account-id:resource-type","resource-id"]`





#### Data type
String
str1 = "Hello"
str2 = "World"
result = str1 + " " + str2
print(result)


text = "   Some spaces around   "
stripped_text = text.strip() -> xóa khoảng trắng
print("Stripped text:", stripped_text)

text = "Python is awesome"
substring = "is"
if substring in text: #dùng if để check 1 string có trong 1 string khác không bằng "in"
    print(substring, "found in the text")

Float
Integer

- List: là collection có thứ tự. List là mutable (có thể thay đổi, thêm, xóa item khỏi list). List được define bằng 2 dấu []
VD define 1 list: my_list = [1, 2, 3, 4, 5, 6]
Sửa phần tử 0 của list: my_list[0] = 10 
print("Modified List:", my_list)
THêm 1 phần tử vào list: my_list.append(10)

- Tuple: là collection có thứ tự. Tuple là immutable (không thể thay đổi, thêm, xóa item). Tuple được define bằng 2 dấu ()
VD define 1 tuple: my_tuple = (1, 2, 3, 4, 5, 6, 7)
db_config=("localhost",5432,"admin","password")
my_tuple[1] = 10 -> sẽ báo lỗi

- Set: là collection không có thứ tự, item trong set là unique (nếu ta define 1 item nhiều lần thì chỉ có 1 item được in ra). Set là mutable (có thể thêm, xóa item nhưng không change được item). Set được define bằng {}
VD define 1 set: my_sets = {1, 2, 3, 4, 5, 6, 1}
print("My Sets:", my_sets) -> kết quả là my_sets = {1,2,3,4,5,6}
Thêm 1 item vào set : my_sets.add(9)
THêm nhiều item vào set: my_set.update([7,8])
Xóa phần tử khỏi set: remove
Set phù hợp cho list mà ta muốn các phần tử không trùng nhau (set sẽ tự eliminate item trùng), VD danh sách IP các server hoặc user group

- Dictionary là collection không có thứ tự của key-value pairs. DIctionary là mutable (add,update, delete key-value pairs). Lưu ý là chỉ có thể thay value, không thể modify key
  - key: phải là unique và immutable (không thể thay đổi key). Key có thể type string, number hoặc tuple
  - value: có thể là any data type (string, list, other dictionary). ** xem lại vd trong vở ansible **
 
VD:
employee = {
  "name": "john doe",
  "age": 30,
  "position": "engineer",
}

In ra giá trị:
print(employee["name"]) 
THêm key-value pair department: IT
employee["department"] = [IT]
Update 1 giá trị :
employee["age"] = 31


- Boolean: là true hoặc false
VD: is_ranining = true -> đặt cho 1 biến là True (true ở đây là datatype chứ không phải string "true")
print(is_raining) -> Output là True

VD:
is_server_up = true
if is_server_up:
  print("The server is running")

VD:
a = 10, b = 20
is_equal = a == b
print(is_equal) -> Output là false


---

### Regular expression (regex)
Dùng để match pattern và string
Ứng dụng: search error log trong log file,...

Để sử dụng regular expression cần import "re" module

Các basic re function
- re.match(): xác định regex match ở đầu string (determines if the regex matches at the beginning of the string)

Dùng re.match() để tìm match ở đầu trong 1 string
import re
text = "The quick brown fox"
pattern = r"quick"
match = re.match(pattern, text) -> match sẽ trả về heo định dạng <re.Match object; span=(0,x), match='pattern'> nếu tìm thấy, không tìm thấy sẽ trả về None. Trường hợp này sẽ trả về None do quick không nằm ở đầu
if match: #tuy match trả về theo định dạng <re.Match.....> nhưng có thể dùng match để check trong if -> match có thể là true/false (cần check lại với chatgpt)
    print("Match found:", match.group()) -> match.group() để lấy ra giá trị match
else:
    print("No match")

 
- re.search(): tìm string và trả về matched string đầu tiên (searches the string for a match and returns the first occurrence)

import re
text = "The quick brown fox"
pattern = r"brown"
search = re.search(pattern, text) -> search sẽ trả về heo định dạng <re.Match object; span=(x,y), match='pattern'>
if search:
    print("Pattern found:", search.group())
else:
    print("Pattern not found")

- re.findall(): returns a list of all matches in the string
- re.sub(): replaces mathces with a new string


Common special  characters:
- . : any character (except newline)
- ^ : start of string
- $ : end of string
- * : 0 or more occurrences
- + : 1 or more occurrences
- [] : character set
- | : or
- \ : escapre special characters
- \d : digit
- \w : word character
- \s : white space



	
Day-02/examples/03-regex-replace.py
import re
text = "The quick brown fox jumps over the lazy brown dog"
pattern = r"brown"
replacement = "red"
new_text = re.sub(pattern, replacement, text)
print("Modified text:", new_text)


	
Day-02/examples/03-regex-split.py
import re
text = "apple,banana,orange,grape"
pattern = r","
split_result = re.split(pattern, text)
print("Split result:", split_result)



File note của Tuấn ở google doc
https://docs.google.com/document/d/1gym7z1nqfo3rhLn0GC73IvbYnSgVdJtCZ85mnZN0i4A/edit?tab=t.0
