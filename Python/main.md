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
VD: is_ranining = True -> đặt cho 1 biến là True (true ở đây là datatype chứ không phải string "true")
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



Day-02/examples/01-string-concat.py
str1 = "Hello"
str2 = "World"
result = str1 + " " + str2
print(result)

Day-02/examples/01-string-len.py
text = "Python is awesome"
length = len(text)
print("Length of the string:", length)

Day-02/examples/01-string-lowercase.py
text = "Python is awesome"
uppercase = text.upper()
lowercase = text.lower()
print("Uppercase:", uppercase)
print("Lowercase:", lowercase)

Day-02/examples/01-string-replace.py
text = "Python is awesome"
new_text = text.replace("awesome", "great")
print("Modified text:", new_text)

Day-02/examples/01-string-split.py
text = "Python is awesome"
words = text.split()
print("Words:", words)

Day-02/examples/01-string-strip.py
text = "   Some spaces around   "
stripped_text = text.strip()
print("Stripped text:", stripped_text)

Day-02/examples/01-string-substring.py
text = "Python is awesome"
substring = "is"
if substring in text:
    print(substring, "found in the text")
	
Day-02/examples/02-float.py
# Float variables
num1 = 5.0
num2 = 2.5

# Basic Arithmetic
result1 = num1 + num2
print("Addition:", result1)

result2 = num1 - num2
print("Subtraction:", result2)

result3 = num1 * num2
print("Multiplication:", result3)

result4 = num1 / num2
print("Division:", result4)

# Rounding
result5 = round(3.14159265359, 2)  # Rounds to 2 decimal places
print("Rounded:", result5)

Day-02/examples/02-int.py
# Integer variables
num1 = 10
num2 = 5

# Integer Division
result1 = num1 // num2
print("Integer Division:", result1)

# Modulus (Remainder)
result2 = num1 % num2
print("Modulus (Remainder):", result2)

# Absolute Value
result3 = abs(-7)
print("Absolute Value:", result3)

Day-02/examples/03-regex-findall.py
import re

text = "The quick brown fox"
pattern = r"brown"

search = re.search(pattern, text)
if search:
    print("Pattern found:", search.group())
else:
    print("Pattern not found")
	
Day-02/examples/03-regex-match.py
import re
text = "The quick brown fox"
pattern = r"quick"
match = re.match(pattern, text)
if match:
    print("Match found:", match.group())
else:
    print("No match")
	
Day-02/examples/03-regex-replace.py
import re
text = "The quick brown fox jumps over the lazy brown dog"
pattern = r"brown"
replacement = "red"
new_text = re.sub(pattern, replacement, text)
print("Modified text:", new_text)

Day-02/examples/03-regex-search.py
import re
text = "The quick brown fox"
pattern = r"brown"
search = re.search(pattern, text)
if search:
    print("Pattern found:", search.group())
else:
    print("Pattern not found")
	
Day-02/examples/03-regex-split.py
import re
text = "apple,banana,orange,grape"
pattern = r","
split_result = re.split(pattern, text)
print("Split result:", split_result)



https://docs.google.com/document/d/1gym7z1nqfo3rhLn0GC73IvbYnSgVdJtCZ85mnZN0i4A/edit?tab=t.0
