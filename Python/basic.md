## What is the difference between deep copy and shallow copy in Python?
---
Python provides two ways to copy objects:

- Shallow Copy: creates a new object but does NOT recursively copy nested objects
- Deep Copy: creates a new object AND recursively copies all nested objects

**Shallow Copy**:
A shallow copy creates a new container object (like a new list, dict, etc.), but the elements inside it are references to the same objects as the original.

```python
import copy

list1 = [1, 2, [3, 4]]
list2 = copy.copy(list1)   # shallow copy

list2[2].append(5)

print(list1)   # [1, 2, [3, 4, 5]]
print(list2)   # [1, 2, [3, 4, 5]]
```
- Updating a nested element affects both lists, Because nested objects are shared

**Deep Copy**: A deep copy creates:
- A new container object
- All nested objects are fully copied (recursively)
- No references are shared with the original

```python
list1 = [1, 2, [3, 4]]
list2 = copy.deepcopy(list1)

list2[2].append(5)

print(list1)  # [1, 2, [3, 4]]
print(list2)  # [1, 2, [3, 4, 5]]
```
- Updating a nested object does not affect the original, truly independent copy

**When to Use What?**

Use Shallow Copy when:
- You have a simple object (no deep nesting)
- You want faster copying
- You don’t need full duplication

Use Deep Copy when:
- You have a complex nested structure
- You want full independence between original & copy
- You want to avoid accidental mutations

**Special Note – Immutables**

Integers, strings, tuples, etc., are immutable, so both shallow and deep copy behave similarly for them.

**Summary**

A shallow copy creates a new object but shares nested objects with the original.A deep copy creates a new object and recursively copies all nested objects, so changes in the nested structure do not affect the original.

## Why tuples are faster than lists in Python?
---
Tuples are faster than lists mainly because they are immutable. Immutability gives Python several optimization opportunities.

**Tuples Are Stored More Compactly in Memory**

Because tuples are immutable, Python can store them in a `fixed, compact memory layout`. Lists, however:
- Need extra memory for `future resizing`
- Store pointers to `elements + dynamic capacity`
- Must track `capacity and size separately`
```python
# Tupple memory layout
[1][2][3]

# List memory layout
[ptr1][ptr2][ptr3] + extra allocated empty slots
```
> Tuples = smaller memory footprint
> Smaller memory ⇒ faster access & iteration

**No Need for Dynamic Resizing**
- Lists can grow or shrink, so Python must allocate extra space and occasionally resize the list.
- Resizing operations are expensive.
- Tuples never change in size → no resize overhead.

**Tuples Allow More Optimizations in CPython**

Because Python knows a tuple will never change:
- Certain operations are optimized at C level
- They require fewer CPU instructions
- Hashing is possible for tuples (immutable), so they can be used as dict keys
- Python runtime performs fewer checks

Example:

- List access requires safety checks for mutability
- Tuple access skips many of those checks

**Tuple Iteration Is Faster**

Due to compact layout + fewer mutability checks. Tuples are usually `~20–30% faster`.

**Tuples Don’t Support Mutability Features**

Lists support:
- append()
- insert()
- pop()
- remove()
- extend()
- slicing assignment
- dynamic resizing

Each of these adds overhead to its internal structure.

Tuples don’t need:
- extra metadata
- dynamic size tracking
- reallocation logic

**When speed matters**: Use tuple if… **
- Data is fixed (configuration, coordinates, DB column sets)
- You’re creating millions of small objects (e.g., in loops)
- You’re using them as dictionary keys
- You want less memory usage

**why tuples can be hashed, why lists cannot?**

Tuples are hashable because they are `immutable`; their contents never change, so their hash value remains constant. Lists are unhashable because they are `mutable`; changing their contents would change their hash, which would break dictionary and set behavior.

**Summary**
Tuples are faster than lists because they are immutable.
They use less memory, require fewer operations, and allow CPython to optimize them more aggressively.
This results in faster access, faster iteration, and less overhead compared to lists.


## What is a defaultdict, and how does it prevent key-related errors when initializing dictionary values?
---
A defaultdict is a subclass of the standard Python dict that accepts a callable argument (a function or constructor like list, int, or set) upon initialization.

Behavior: When you try to access a key that is not present in the dictionary, the defaultdict automatically calls the provided function/constructor with no arguments to create a default value for that key, inserts it, and then returns it.

Error Prevention: It prevents the KeyError that would occur when trying to access or append to a non-existent key in a standard dictionary.

```python
from collections import defaultdict

# Default value will be a list()
category_counts = defaultdict(list)

# If 'fruits' didn't exist, this would raise a KeyError in a standard dict.
# Here, defaultdict creates an empty list for 'fruits' before appending.
category_counts['fruits'].append('apple')
category_counts['vegetables'].append('carrot')

print(category_counts['fruits']) # Output: ['apple']
print(category_counts['dairy'])  # Output: [] (A new empty list was created)
```
## First Class functions in Python
---
In Python, functions are treated as `first-class objects`. This means they can be used just like numbers, strings, or any other variable. You can:
- Assign functions to variables.
- Pass them as arguments to other functions.
- Return them from functions.
- Store them in data structures such as lists or dictionaries.

This ability allows you to write reusable, modular and powerful code.

### Characteristics of First-Class Functions
Functions in Python have the following important characteristics. Let’s see them one by one with examples:

**Assigning Functions to Variables**
```python
def msg(name):
    return f"Hello, {name}!"

# Assigning the function to a variable
f = msg

# Calling the function using the variable
print(f("Emma"))
```
** Passing Functions as Arguments**
```python
def msg(name):
    return f"Hello, {name}!"

def fun1(fun2, name):
    return fun2(name)

# Passing the msg function as an argument
print(fun1(msg, "Alex"))
```
**Returning Functions from Other Functions**
```python
def fun1(msg):
    def fun2():
        return f"Message: {msg}"
    return fun2

# Getting the inner function
func = fun1("Hello, World!")
print(func())
```
**Storing Functions in Data Structures**
```python
def add(x, y):
    return x + y

def subtract(x, y):
    return x - y

# Storing functions in a dictionary
d = {
    "add": add,
    "subtract": subtract
}

# Calling functions from the dictionary
print(d["add"](5, 3))       
print(d["subtract"](5, 3))
```


## Higher Order Functions in Python
---
In Python, Higher Order Functions (HOFs) play an important role in functional programming and allow for writing more modular, reusable and readable code. A Higher-Order Function is a function that either:
- Takes another function as an argument
- Returns a function as a result

```python
def greet(func): # higher-order function 
    return func("Hello")

def uppercase(text): # function to be passed
    return text.upper()

print(greet(uppercase))
```
**Explanation**: greet(func) is a higher-order function as it takes another function as an argument. uppercase(text) converts text to uppercase. Calling greet(uppercase) passes "Hello" to uppercase, resulting in "HELLO".