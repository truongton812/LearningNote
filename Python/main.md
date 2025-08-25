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
- String
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
- None: đại diện cho null
- Float
- Integer

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


- Boolean: là True hoặc False (lưu ý phải là chữ in)
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

Use case: tìm log lỗi trong file

import re
pattern = "(WARNING|ERROR)" #tìm pattern có chứa WARNING hoặc ERROR
file_path = "/var/log/messages"
with open(file_path, "r") as file: #đọc file bằng with open()
  for line in file:
    if re.search(pattern, line):
	  print(line.strip()) #print matching lines without extra spaces

--

### Regular expression (regex)
Dùng để match pattern và string
Ứng dụng: search error log trong log file,...

Để sử dụng regular expression cần import "re" module

Các basic re function:

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

import re
text = "hi hello hi hello hi"
pattern = r"hi"
result = re.findall(pattern, text) -> kết quả sẽ in ra ['hi', 'hi', 'hi'] (trông hơi vô dụng?)



- re.sub(): replaces matches with a new string
VD1:
import re
text = "The quick brown fox jumps over the lazy brown dog"
pattern = r"brown"
replacement = "red"
new_text = re.sub(pattern, replacement, text)
print("Modified text:", new_text)

VD2: thay thế nhiều khoảng trắng thành 1 khoảng trắng
text = "Python     is      great"
pattern = "\s+" #one or more spaces
new_string = re.sub(pattern, " ", text)

- re.split(): dùng để chia text bằng các delimeter

import re
text = "apple,banana,orange,grape"
pattern = r","
split_result = re.split(pattern, text)
print("Split result:", split_result) -> output là ['apple','banana','orange','grape']



Lưu ý: trong các hàm trên, thay vì đặt thành biến, có thể đưa thẳng vào trong hàm
VD: new = re.sub("brown", "red", "The quick brown fox jumps over the lazy brown dog")
Lưu ý 2: pattern chỉ cần đặt là pattern="abc", không hiểu sao trong bài giảng cần r ở trước

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



	
### keywords

keyword là những word được định nghĩa sẵn trong python. 

if / elif / else: Used for conditional statements.
for / while: Used for looping. for thường dùng để lặp qua các phần tử trong danh sách, tuple, v.v. while lặp lại cho đến khi một điều kiện trở nên sai.
break / continue: Used to control loop execution. break dùng để dừng vòng lặp ngay lập tức. continue bỏ qua phần còn lại trong lần lặp hiện tại và chuyển sang lần lặp tiếp theo.
pass: A placeholder statement that does nothing. pass: Là lệnh không làm gì cả, thường dùng như một placeholder khi chưa muốn viết mã vào đó.
return: Exits a function and returns a value. Kết thúc một hàm và trả về một giá trị cho nơi gọi hàm.
yield: Returns a value in a generator function and pauses its state. Dùng trong hàm generator để trả về giá trị tạm thời, hàm có thể tạm dừng và tiếp tục lại tại vị trí đó trong lần gọi sau.
try / except / finally: Used for exception handling. Dùng để bắt và xử lý lỗi, ngăn chương trình dừng đột ngột. finally luôn được thực hiện dù có lỗi hay không.
raise: Used to raise an exception. Dùng để chủ động phát sinh một lỗi (exception) trong chương trình.


logical: and / or / not
function và class: def, class, lambda
variable scope và namespace: global, nonlocal (refers to variable in the nearest enclosing scope, excluding globals)
importing và module: import/from , as(dùng để give an imported module or function an alias)
OOP: self, is, in/not in
miscellaneous:assert, del, with

---

1 -python_scripts.txt
In Python, keywords are reserved words that have special meanings and cannot be used as variable names. Here's a list of commonly used Python keywords with small examples for each:

1. if
Used for conditional statements.

x = 10
if x > 5:
    print("x is greater than 5")
2. else
Used with if to provide an alternative.

x = 2
if x > 5:
    print("x is greater than 5")
else:
    print("x is less than or equal to 5")

3. elif
Short for "else if", used to check multiple conditions.

x = 5
if x > 5:
    print("x is greater than 5")
elif x == 5:
    print("x is equal to 5")
else:
    print("x is less than 5")

4. for
Used for looping over a sequence.

for i in range(3):
    print(i)

5. while
Used for looping while a condition is true.

i = 0
while i < 3:
    print(i)
    i += 1

6. break
Used to exit a loop.

for i in range(5):
    if i == 3:
        break
    print(i)

7. continue
Used to skip the rest of the loop and continue with the next iteration.

for i in range(5):
    if i == 3:
        continue
    print(i)

8. def
Used to define a function.

def greet():
    print("Hello, world!")
greet()

9. return
Used to return a value from a function.

def add(a, b):
    return a + b
result = add(2, 3)
print(result)

10. class
Used to define a class in Python.

class Dog:
    def bark(self):
        print("Woof!")

dog = Dog()
dog.bark()

11. import
Used to import modules.

import math
print(math.sqrt(16))

12. try, except
Used for exception handling.

try:
    result = 10 / 0
except ZeroDivisionError:
    print("You can't divide by zero!")

13. lambda
Used to create anonymous functions.

add = lambda x, y: x + y
print(add(2, 3))

14. pass
Used to skip a block of code.

def my_function():
    pass  # To be implemented later

15. with
Used to simplify exception handling when working with resources like files.

with open('test.txt', 'w') as f:
    f.write('Hello, world!')

16. in
Used to check for membership in a sequence.

numbers = [1, 2, 3, 4]
if 3 in numbers:
    print("3 is in the list")

17. not
Logical NOT operator.

x = False
if not x:
    print("x is False")

18. and
Logical AND operator.

x = True
y = False
if x and not y:
    print("x is True and y is False")

19. or
Logical OR operator.

x = False
y = True
if x or y:
    print("At least one is True")

20. is
Used to check if two variables refer to the same object.

a = [1, 2, 3]
b = a
if a is b:
    print("a and b refer to the same object")

21. None
Represents the absence of a value.

x = None
if x is None:
    print("x is None")
These are just a few examples of Python keywords, which help define the structure and control the flow of a program.


	





File note của Tuấn ở google doc
https://docs.google.com/document/d/1gym7z1nqfo3rhLn0GC73IvbYnSgVdJtCZ85mnZN0i4A/edit?tab=t.0
