How to Declare an Array in Bash
Declaring an array in Bash is easy, but pay attention to the syntax. If you are used to programming in other languages, the code might look familiar, but there are subtle differences that are easy to miss.

To declare your array, follow these steps:

Give your array a name
Follow that variable name with an equal sign. The equal sign should not have any spaces around it
Enclose the array in parentheses (not brackets like in JavaScript)
Type your strings using quotes, but with no commas between them
Your array declaration will look something like this:

myArray=("cat" "dog" "mouse" "frog")
That's it! It's that simple.

How to Access an Array in Bash
There are a couple different ways to loop through your array. You can either loop through the elements themselves, or loop through the indices.

How to Loop Through Array Elements
To loop through the array elements, your code will need to look something like this:

for str in ${myArray[@]}; do
  echo $str
done
To break that down: this is somewhat like using forEach in JavaScript. For each string (str) in the array (myArray), print that string.

The output of this loop looks like this:

cat
dog
mouse
frog
Note: The @ symbol in the square brackets indicates that you are looping through all of the elements in the array. If you were to leave that out and just write for str in ${myArray}, only the first string in the array would be printed.

How to Loop Through Array Indices
Alternatively, you can loop through the indices of the array. This is like a for loop in JavaScript, and is useful for when you want to be able to access the index of each element.

To use this method, your code will need to look something like the following:

for i in ${!myArray[@]}; do
  echo "element $i is ${myArray[$i]}"
done
The output will look like this:

element 0 is cat
element 1 is dog
element 2 is mouse
element 3 is frog
Note: The exclamation mark at the beginning of the myArray variable indicates that you are accessing the indices of the array and not the elements themselves. This can be confusing if you are used to the exclamation mark indicating negation, so pay careful attention to that.

Another note: Bash does not typically require curly braces for variables, but it does for arrays. So you will notice that when you reference an array, you do so with the syntax ${myArray}, but when you reference a string or number, you simply use a dollar sign: $i.



Key Operations with Indexed Arrays
The examples below use the array my_array=("apple" "banana" "cherry"). 
Operation 	Syntax	Example	Description
Declaration	name=(item1 item2 ...)	my_array=("apple" "banana" "cherry")	Creates an array with initial values. Spaces around the equal sign are not allowed.
Explicit Declaration	declare -a name	declare -a my_array	Declares an empty indexed array.
Access Element	${name[index]}	echo ${my_array[0]}	Prints the element at the specified index (0-based).
Access All	${name[@]} or ${name[*]}	echo ${my_array[@]}	Prints all elements of the array.
Array Size	${#name[@]} or ${#name[*]}	echo ${#my_array[@]}	Returns the number of elements in the array.
Add Element	name+=(item)	my_array+=("date")	Appends a new element to the end of the array.
Modify Element	name[index]=value	my_array[1]="blueberry"	Replaces the value at a specific index.
Remove Element	unset name[index]	unset my_array[1]	Deletes a single element at the specified index.
Remove Entire Array	unset name	unset my_array	Deletes the entire array variable.
Iterating through an Array
You can loop through array elements using a for loop: 
bash
for item in "${my_array[@]}"; do
  echo "Fruit: $item"
done
This method ensures that elements containing spaces are handled correctly by using double quotes and the @ symbol. 
Associative Arrays
Associative arrays use arbitrary strings as keys (like key-value pairs) instead of integer indices. 
Declaration: declare -A associative_array (This is mandatory).
Assignment: associative_array[key]="value"
Example:
bash
declare -A fruits
fruits[south]="Banana"
fruits[north]="Orange"
echo ${fruits[south]} # Output: Banana
To loop through the keys of an associative array, use ${!name[@]}:
bash
for key in "${!fruits[@]}"; do
  echo "Key: $key, Value: ${fruits[$key]}"
done
