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
- len(arrray): đến số phần tử của chuỗi
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

- Set: là collection không có thứ tự, item trong set là unique (nếu ta define 1 item nhiều lần thì set sẽ tự remove các duplicate item đấy đi). Set là mutable (có thể thêm, xóa item nhưng không change được item). Set được define bằng {}
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

- None Variable
None is a special constant in Python that represents the absence of a value or a null value.


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
```
import re
text = "The quick brown fox"
pattern = r"brown"
search = re.search(pattern, text) -> search sẽ trả về heo định dạng <re.Match object; span=(x,y), match='pattern'>
if search:
    print("Pattern found:", search.group())
else:
    print("Pattern not found")
```

```
Use case: tìm log lỗi trong file

import re
pattern = "(WARNING|ERROR)" #tìm pattern có chứa WARNING hoặc ERROR
file_path = "/var/log/messages"
with open(file_path, "r") as file: #đọc file bằng with open()
  for line in file:
    if re.search(pattern, line):
	  print(line.strip()) #print matching lines without extra spaces
```

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

- if / elif / else: Used for conditional statements.
```
if x > 5: #1
    print("x is greater than 5") #thực thi khi #1 true
elif x == 5: #2
    print("x is equal to 5") #thực thi khi #1 false và #2 true
else: #3
    print("x is less than 5") #thực thi khi #1 and #2 false, #3 true
```


- for / while: Used for looping. for thường dùng để lặp qua các phần tử trong danh sách, tuple, v.v. while lặp lại cho đến khi một điều kiện trở nên sai.
```
for i in range(3):
    print(i)
```

```
i = 0
while i < 3:
  print(i)
  i += i
```

- break / continue: Used to control loop execution. break dùng để dừng vòng lặp ngay lập tức. continue bỏ qua phần còn lại trong lần lặp hiện tại và chuyển sang lần lặp tiếp theo.

```
for i in range(5):
  if i == 3:
    break
  print(i)
```
-> kết quả là in ra 0 1 2 
```
for i in range(5):
  if i == 3:
    continue
  print(i)
```
-> kết quả là in ra 0 1 2 4


- pass: A placeholder statement that does nothing. pass: Là lệnh không làm gì cả, thường dùng như một placeholder khi chưa muốn viết mã vào đó.
- return: Exits a function and returns a value. Kết thúc một hàm và trả về một giá trị cho nơi gọi hàm.

```
def add(a, b):
    return a + b
result = add(2, 3)
print(result)
```

- yield: Returns a value in a generator function and pauses its state. Dùng trong hàm generator để trả về giá trị tạm thời, hàm có thể tạm dừng và tiếp tục lại tại vị trí đó trong lần gọi sau.
- try / except / finally: Used for exception handling. Dùng để bắt và xử lý lỗi, ngăn chương trình dừng đột ngột. finally luôn được thực hiện dù có lỗi hay không.

```
try:
    result = 10 / 0
except ZeroDivisionError:
    print("You can't divide by zero!")
```
- raise: Used to raise an exception. Dùng để chủ động phát sinh một lỗi (exception) trong chương trình.

- with: Used to simplify exception handling when working with resources like files.
```
with open('test.txt', 'w') as f:
    f.write('Hello, world!')
```

- logical: and / or / not
- function và class: def, class, lambda
```
def greet():
    print("Hello, world!")
greet()
```
```
class Dog:
    def bark(self):
        print("Woof!")

dog = Dog()
dog.bark()
```

variable scope và namespace: global, nonlocal (refers to variable in the nearest enclosing scope, excluding globals)
importing và module: import/from , as(dùng để give an imported module or function an alias)
OOP: self, is, in/not in
miscellaneous:assert, del, with

---



- in: Used to check for membership in a sequence.
```
numbers = [1, 2, 3, 4]
if 3 in numbers:
    print("3 is in the list")
```
- not: Logical NOT operator.
```
x = False
if not x:
    print("x is False")
```
- and: Logical AND operator.
```
x = True
y = False
if x and not y:
    print("x is True and y is False")
```

- or: Logical OR operator.

```
x = False
y = True
if x or y:
    print("At least one is True")
```

- is: Used to check if two variables refer to the same object.

a = [1, 2, 3]
b = a
if a is b:
    print("a and b refer to the same object")

- None : Represents the absence of a value.
```
x = None
if x is None:
    print("x is None")
```

### Chi tiết hơn về loop
Loop trong python có 2 loại là for loop (dùng để iterates over a sequence (như là list, tuple, string hoặc range)). While to repeat a block of code until a certain condition is met

VD: languagues = ['Swift', 'Python', 'Go']
for lang in languagues:
  print(lang)

hoặc for in range(1, 10) #loop từ 1 đến 9
hoặc text = "Python is a program". for i in text: #loop trên string


VD: number = 1
while number <= 3:
  print(number)
  number += 1
Lưu ý khi tạo while loop cần phải có cách để thoát vòng lặp, nếu ko sẽ loop vô hạn


Break statement trong python
Break statement dùng để thoát loop sớm hơn dự kiến khi có condition thỏa mãn

VD:
```
i = 0
while True:
  print(i)
  i += 1
  if i == 5:
    break
```

### Working with variables

Trong python, để gọi variable đơn giản chỉ cần gọi đến tên variable
Nếu muốn gọi variable trong dấu double quote thì dùng dấu ``{}``
In Python, variables are used to store data that can be used and manipulated throughout a program. Unlike other programming languages, Python doesn't require you to explicitly declare the variable type, as it is dynamically typed.

In python, local and global variables refer to the scope in which a variable is defined and can be accessed

- Local variables are those defined inside the function and are only accessible within the function

```
def addition():
    num1 = 40
    num2 = 30
    print ( num1 + num2)

addition()

def subs():
    num1 = 50
    num2 = 25
    print ( num1 - num2)

subs()

#2 function sử dụng cùng 2 var name num1 và num2 nhưng giá trị khác nhau, do chỉ có ý nghĩa local
```

- Global variables are those defined outside any function and are accessible throughtout the code, including inside function
```
num1 = 100
num2 = 30

def addition():
    num1 = 50
    num2 = 20
    print ( num1 + num2)

addition()

def subs():
    print ( num1 - num2) #không cần define variable trong function vẫn có thể dùng được từ global

subs()
```

Lưu ý local variable sẽ ghi đè global variable trong function

Note: function là block of code, chỉ được thực thi khi ta gọi đến
VD:

def my_function(): #define function 
    print("Hello from a function")
my_function() #gọi function



 
### Return statement 
Return statement được dùng để exit a function and send back a value to the caller

When a function is called, the code inside the function is executed, and when the return statement is encountered, the function stops executing and returns the specified value to the caller. If no value is specified, the function returns None by default.

Lưu ý: khi return cái gì mà muốn hiển thị ra thì phải print ra, chứ bình thường return không hiển thị ra output

Syntax:
```
def function_name(parameters):
#Code
return value
```

VD:


def add(a, b):
    return a + b
	print ("hello") #do return exit function nên dòng này sẽ không được thực thi

result = add(5, 5)
print(result) #output: 8



- Returning Multiple Values
```
def get_details():
    name = "Jai"
    age = 30
    return name, age
    
details = get_details()
print(details) #kết quả trả về là tuple
```

- return theo điều kiện

```
def check_even(number):
    if number % 2 == 0:
        return "Even"
    return "Odd" #đỡ được một bước else

result = check_even(8)
print(result)

```


### Function

In Python, a function is a reusable block of code that performs a specific task.

Functions help to organize and manage code by encapsulating tasks into manageable sections.

Functions can take inputs (parameters), perform operations, and return outputs.

Có thể gọi function trong function

SYntax:
def function_name(parameter):
  #code
  return
Trong đó parameter và return là optional
Cách gọi function: function_name()

VD:
```
#Function with Parameters
def greet(name, age):
    print(f"Hello, {name} ! you are {age} years old.")

# calling the function with arguments

greet("Shikhar", 35)
```

Lưu ý Chữ "f" trong đoạn code Python trên là phần khai báo một chuỗi định dạng (gọi là f-string hay formatted string literal). Khi một chuỗi được đặt trước bởi chữ "f" (hay "F"), các biểu thức bên trong dấu ngoặc nhọn {} sẽ được tính toán và giá trị của chúng sẽ được chèn trực tiếp vào chuỗi kết quả khi chạy chương trình.


### Module
Module trong python là để tái sử dụng code giữa các app.py. Module là single python file, có thể chứa function, class, variable
Các builtin module math, random, datetime, os, sys, json, re, time (tra chatgpt để biết công dụng từng module)
Ngoài ra còn có thể tạo custom module

Muốn dùng module thì phải import trước

Cách gọi function trong module: <module_name>.<function_name>

VD builtin module

```
# Get the current working directory
current_directory = os.getcwd()
print(f"Current working directory: {current_directory}")

# list the files and directories
files = os.listdir(".")
print(f"Files in the current directory: {files}")
```


VD custom module

```
# mymodule.py

def greet(name):
    return f"Hello, {name}"

def add(a, b):
    return a + b

my_variable = 45

```

```
# Import the user-defined module
import mymodule

# Use the functions and variables from the module (mymodule)
name = "Alice"
greeting = mymodule.greet(name)
print(greeting)

#using the add function
result = mymodule.add(10, 10)
print(f"10 + 10 = {result}")
```




### Package

Package là cách để organize các related module thành 1 thư mục có cấu trúc

Package giúp quản lý large codebases. Packages cho phép group các modules có chung tính năng theo logical. Mỗi package là 1 namespace riêng, giúp không bị conflict identifier in different module (còn thiếu 1 vài benifit, hỏi thêm chatgpt)

Package: Package về cơ bản là một thư mục chứa nhiều module và một file đặc biệt tên là init.py. File init.py giúp nhận diện thư mục đó là một package Python. Trong package có thể có subpackage

Ví dụ cấu trúc package:
```
my_package/
│
├── __init__.py         # Biến thư mục này thành package
├── module1.py          # Một module Python (file)
├── module2.py          # Một module Python khác
│
└── sub_package/
    │
    ├── __init__.py     # Biến thư mục này thành subpackage
    └── submodule1.py   # Module trong subpackage

```


Ví dụ cách tạo và sử dụng package trong code

Cấu trúc thư mục ví dụ
```
package_folder/
├── main.py
└── my_package/
    ├── __init__.py    
    └── math_operations.py
    └── string_operations.py
```
Trong đó

```
math_operations.py

def add(a, b):
    return a + b

def subtract(a, b):
    return a - b
```

```
string_operations.py

def to_upper(s):
    return s.upper()

def to_lower(s):
    return s.lower()
```

```
__init__.py #thấy trong bài giảng bảo file này có thể để trống

from .math_operations import add, subtract #cái này có vẻ khác khi import function ở trên. Ở đây ta từ module import function, ở trên là import thẳng module. Dấu . ở đây là do module nằm cùng cấp với file init
from .string_operations import to_upper, to_lower
```

Sử dụng module từ package vào trong file main.py

```
main.py

# Import the package (my_package)
import my_package

# Use functions from the package

result_add = my_package.add(3, 5)
result_subtract = my_package.subtract(10, 2)
result_upper = my_package.to_upper("hello")
result_lower = my_package.to_lower("WORLD")

print(f"Add: {result_add}")
print(f"Subtract: {result_subtract}")
print(f"Uppercase: {result_upper}")
print(f"Lowercase: {result_lower}")
```


Nói chung xong không hiểu dùng package thì lợi gì hơn dùng module. Sau làm lab lại


### Command line arguments
Dùng để pass argument cho chương trình python khi chạy từ terminal

VD: python script.py arg1 arg2 arg3

Để access argument trong python thì ta dùng sys module
Trong đó sys.argv[0] là script name, sys.argv là tất cả các parameter truyền vào (trong đó có cả script name. Tức sys.argv là gồm sys.argv[0], sys.argv[1], sys.argv[2],...)
sys.argv[1], sys.argv[2],... từng là parameter truyền vào

Ví dụ

```
s3.py

import sys

#Writing functions for add, sub & multiplication

def add(num1, num2):
    return num1 + num2

def sub(num1, num2):
    return num1 - num2

def mult(num1, num2):
    return num1 * num2

num1 = int(sys.argv[1])
operator = sys.argv[2]
num2 = int(sys.argv[3])

if operator == 'add':
    output = add(num1, num2)
    print (output)
elif operator == 'sub':
    output = sub(num1, num2)
    print (output)
elif operator == 'mult':
    output = mult(num1, num2)
    print(output)
else: 
    print("Please give the correct operator, it is add, sub, multi")
	
	
===
s4.py

import sys

# Print all command-line arguments
print("Arguments passed:", sys.argv)

# First argument is always the script name
script_name = sys.argv[0]
print("Script name:", script_name)

# Other arguments are passed from index 1 onward
if len(sys.argv) > 1: #len của array là số lượng phần tử trong chuỗi
    first_argument = sys.argv[1]
    print("First argument:", first_argument)
else:
    print("No arguments passed.")
```



### Operator trong python

- arithmetic operator: + , - , * , / , % (lấy phần dư), ** (căn bậc), // (chia lấy phần nguyên)
VD: 10 % 3 = 1 , 2 ** 3 = 8 , 10 // 3 = 3

- comparision (relational) operator: giúp so sánh 2 value và trả về true hoặc false
== , != , > , < , >= , <=

- logical operator: dùng để combine conditional statement hoặc expression, trả về true hoặc false
and, or, not
VD not (a < b)

- assignment operator: dùng để assign value cho biến
 + =
 + += . VD x += 5 (tương đương x = x + 5)
 + -= . VD x -= 5 (tương đương x = x -5 )
 + *= . VD x *= 5
 + /= . VD x /= 5
 + %= . VD x %= 5
 + **= . VD x **= 5
 + //= . VD x //= 5

VD: 
```
timeout = 30
load_factor = 2
timeout *= load_factor #adjust the timeout
```

- membership operator: dùng để check xem 1 value có phải là phần tử của sequence (VD string/list/tuple hoặc set) không.
  - in : trả về true nếu value có trong sequence
  - not in: trả về true nếu value không có trong sequence
 
 VD: fruits = ['apple', 'banana', 'cheery']
 print('banana' in fruits) -> output sẽ là true
 print('grape' in fruits) -> output sẽ là false
 print('grape' not in fruits) -> output sẽ là true

 ### List và exception handling

File note của Tuấn ở google doc
https://docs.google.com/document/d/1gym7z1nqfo3rhLn0GC73IvbYnSgVdJtCZ85mnZN0i4A/edit?tab=t.0
  
