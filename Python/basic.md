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

**Ways to create copies in Python:**

| Method | Type | Works with |
| --- | --- | --- |
| `copy.copy(obj)` | Shallow | Any object |
| `copy.deepcopy(obj)` | Deep | Any object |
| `list(original)` | Shallow | Lists |
| `original[:]` | Shallow | Lists |
| `original.copy()` | Shallow | Lists, dicts, sets |
| `dict(original)` | Shallow | Dicts |
| `{**original}` | Shallow | Dicts |

**Visual representation:**
```
Shallow Copy:                     Deep Copy:

original ──► [1, 2, [3, 4]]      original ──► [1, 2, [3, 4]]
                      │
copy     ──► [1, 2,   ↑  ]       copy     ──► [1, 2, [3, 4]]  ← independent!
              (shared ref)                      (separate object)
```

**Interview follow-up: What about `=` assignment?**
```python
a = [1, 2, 3]
b = a        # NOT a copy! Both point to the same object
b.append(4)
print(a)     # [1, 2, 3, 4] — both affected
print(a is b)  # True — they ARE the same object
```
Assignment (`=`) never copies. It creates another reference to the same object.

**Summary**

A shallow copy creates a new object but shares nested objects with the original. A deep copy creates a new object and recursively copies all nested objects, so changes in the nested structure do not affect the original.

---

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

**Benchmark: Tuple vs List creation**
```python
import timeit

# Creating a tuple vs list with the same elements
print(timeit.timeit('(1, 2, 3, 4, 5)', number=1_000_000))
# ~0.01 seconds

print(timeit.timeit('[1, 2, 3, 4, 5]', number=1_000_000))
# ~0.05 seconds (5x slower)
```

**Memory comparison:**
```python
import sys

t = (1, 2, 3, 4, 5)
l = [1, 2, 3, 4, 5]

print(sys.getsizeof(t))  # 80 bytes
print(sys.getsizeof(l))  # 104 bytes (30% more)
```

**Tuple caching (tuple interning):**

CPython caches small tuples and reuses them. This is an optimization not available for lists.
```python
a = (1, 2, 3)
b = (1, 2, 3)
print(a is b)  # True — same object in memory (cached!)

c = [1, 2, 3]
d = [1, 2, 3]
print(c is d)  # False — different objects
```

**Quick comparison table:**

| Feature | Tuple | List |
| --- | --- | --- |
| Mutable | No | Yes |
| Memory usage | Less | More |
| Creation speed | Faster | Slower |
| Iteration speed | ~20-30% faster | Slower |
| Hashable | Yes (if contents are hashable) | No |
| Can be dict key | Yes | No |
| Can be set element | Yes | No |
| Supports `append`, `pop`, etc. | No | Yes |

**Summary**
Tuples are faster than lists because they are immutable.
They use less memory, require fewer operations, and allow CPython to optimize them more aggressively.
This results in faster access, faster iteration, and less overhead compared to lists.

---

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

**Common default factories:**

| Factory | Default value | Typical use case |
| --- | --- | --- |
| `list` | `[]` | Grouping items by key |
| `int` | `0` | Counting occurrences |
| `set` | `set()` | Collecting unique items per key |
| `float` | `0.0` | Accumulating numeric values |
| `str` | `""` | Building strings per key |
| `lambda: None` | `None` | Custom sentinel value |

**Practical example — counting word frequency:**
```python
from collections import defaultdict

text = "apple banana apple cherry banana apple"
word_count = defaultdict(int)

for word in text.split():
    word_count[word] += 1

print(dict(word_count))  # {'apple': 3, 'banana': 2, 'cherry': 1}
```

**Practical example — grouping:**
```python
from collections import defaultdict

students = [
    ("Alice", "Math"),
    ("Bob", "Science"),
    ("Alice", "Science"),
    ("Bob", "Math"),
]

courses = defaultdict(list)
for name, course in students:
    courses[name].append(course)

print(dict(courses))
# {'Alice': ['Math', 'Science'], 'Bob': ['Science', 'Math']}
```

**defaultdict vs `dict.setdefault()`:**
```python
# Using setdefault (standard dict)
d = {}
d.setdefault('fruits', []).append('apple')  # works but verbose

# Using defaultdict (cleaner)
from collections import defaultdict
d = defaultdict(list)
d['fruits'].append('apple')  # cleaner and faster
```

**Interview tip:** `defaultdict` is slightly faster than `dict.setdefault()` because the default factory is called in C, while `setdefault()` involves Python-level method calls.

---

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

**Why does this matter? Real-world uses of first-class functions:**

| Use case | How first-class functions enable it |
| --- | --- |
| **Decorators** | Functions that take a function and return a modified function |
| **Callbacks** | Pass a function to run when an event occurs |
| **Strategy pattern** | Store different algorithms as functions, pick at runtime |
| **Map/Filter/Reduce** | Pass a transform function to built-in operations |
| **Closures** | Return a function that "remembers" its enclosing scope |
| **Sorting** | `sorted(data, key=some_function)` |

**Example — functions as sorting keys:**
```python
students = [
    {"name": "Alice", "gpa": 3.8},
    {"name": "Bob", "gpa": 3.5},
    {"name": "Charlie", "gpa": 3.9},
]

# Pass a function as the key argument
sorted_students = sorted(students, key=lambda s: s["gpa"], reverse=True)
print([s["name"] for s in sorted_students])  # ['Charlie', 'Alice', 'Bob']
```

**Interview tip:** In Python, everything is an object — including functions. Functions have attributes like `__name__`, `__doc__`, `__code__`, etc.
```python
def greet():
    """Say hello"""
    pass

print(type(greet))       # <class 'function'>
print(greet.__name__)    # greet
print(greet.__doc__)     # Say hello
```

---

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

**Built-in Higher Order Functions in Python:**

### `map()` — Apply a function to every item
```python
numbers = [1, 2, 3, 4, 5]
squared = list(map(lambda x: x ** 2, numbers))
print(squared)  # [1, 4, 9, 16, 25]
```

### `filter()` — Keep items where function returns True
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8]
evens = list(filter(lambda x: x % 2 == 0, numbers))
print(evens)  # [2, 4, 6, 8]
```

### `reduce()` — Accumulate items into a single result
```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]
total = reduce(lambda acc, x: acc + x, numbers)
print(total)  # 15
```

### `sorted()` with `key` — Custom sorting
```python
words = ["banana", "apple", "cherry"]
result = sorted(words, key=len)
print(result)  # ['apple', 'banana', 'cherry']
```

**`map`/`filter` vs list comprehensions:**
```python
# map + filter
result = list(map(lambda x: x**2, filter(lambda x: x % 2 == 0, range(10))))

# Equivalent list comprehension (more Pythonic)
result = [x**2 for x in range(10) if x % 2 == 0]

# Both produce: [0, 4, 16, 36, 64]
```

**Interview tip:** In modern Python, list comprehensions are generally preferred over `map`/`filter` for readability. Use `map`/`filter` when passing an existing named function:
```python
names = ["  alice  ", "  bob  "]
cleaned = list(map(str.strip, names))  # cleaner than a lambda
```

---

## What are `*args` and `**kwargs`?
---
`*args` and `**kwargs` allow functions to accept a variable number of arguments.

- `*args` — collects extra **positional** arguments as a **tuple**
- `**kwargs` — collects extra **keyword** arguments as a **dictionary**

```python
def demo(*args, **kwargs):
    print(f"args: {args}")       # tuple
    print(f"kwargs: {kwargs}")   # dict

demo(1, 2, 3, name="Alice", age=25)
# args: (1, 2, 3)
# kwargs: {'name': 'Alice', 'age': 25}
```

**Order of parameters:**
```python
def func(regular, *args, keyword_only, **kwargs):
    pass

# The order MUST be:
# 1. Regular positional arguments
# 2. *args
# 3. Keyword-only arguments (after *args)
# 4. **kwargs
```

**Practical example — flexible logging:**
```python
def log(level, message, *tags, **metadata):
    print(f"[{level}] {message}")
    if tags:
        print(f"  Tags: {', '.join(tags)}")
    for key, value in metadata.items():
        print(f"  {key}: {value}")

log("ERROR", "Database connection failed",
    "db", "critical",
    server="prod-01", retry_count=3)

# [ERROR] Database connection failed
#   Tags: db, critical
#   server: prod-01
#   retry_count: 3
```

**Unpacking with `*` and `**`:**
```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))          # 6 — unpacks list into positional args

params = {"a": 1, "b": 2, "c": 3}
print(add(**params))       # 6 — unpacks dict into keyword args
```

---

## Mutable vs Immutable Objects
---
Every object in Python is either **mutable** (can be changed) or **immutable** (cannot be changed after creation).

| Immutable | Mutable |
| --- | --- |
| `int`, `float`, `bool` | `list` |
| `str` | `dict` |
| `tuple` | `set` |
| `frozenset` | `bytearray` |
| `bytes` | Custom objects (by default) |

**Why does this matter?**

```python
# Immutable — "changing" creates a new object
a = "hello"
b = a
a = a + " world"   # new string object created
print(b)           # "hello" — unchanged
print(a is b)      # False — different objects now

# Mutable — changing modifies the same object
x = [1, 2, 3]
y = x
x.append(4)
print(y)           # [1, 2, 3, 4] — also changed!
print(x is y)      # True — same object
```

**The dangerous default argument trap:**
```python
# WRONG — mutable default argument shared across all calls
def add_item(item, items=[]):
    items.append(item)
    return items

print(add_item("a"))  # ['a']
print(add_item("b"))  # ['a', 'b']  ← BUG! Expected ['b']

# CORRECT — use None as default
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

**`id()` and `is` vs `==`:**
```python
a = [1, 2, 3]
b = [1, 2, 3]

print(a == b)   # True  — same value
print(a is b)   # False — different objects in memory
print(id(a))    # e.g., 140234567890
print(id(b))    # e.g., 140234567950 (different)

# But with small integers (Python caches -5 to 256):
x = 42
y = 42
print(x is y)   # True — same cached object
```

---

## What is the GIL (Global Interpreter Lock)?
---
The GIL is a **mutex** (lock) in CPython that allows only **one thread** to execute Python bytecode at a time, even on multi-core processors.

**Why does the GIL exist?**
- CPython's memory management (reference counting) is not thread-safe
- The GIL prevents race conditions on reference counts
- It simplifies CPython's implementation

**Impact:**
```
With GIL (CPython):
  Thread 1: ████░░░░████░░░░████   (runs, waits, runs, waits...)
  Thread 2: ░░░░████░░░░████░░░░   (alternates with thread 1)
  → Only 1 thread executes at a time

True parallelism (multiprocessing):
  Process 1: ████████████████████   (runs independently)
  Process 2: ████████████████████   (runs independently)
  → Both execute simultaneously on different cores
```

**When the GIL matters and when it doesn't:**

| Scenario | Impact | Solution |
| --- | --- | --- |
| **CPU-bound** (math, data processing) | GIL is a bottleneck — threads don't help | Use `multiprocessing` or `concurrent.futures.ProcessPoolExecutor` |
| **I/O-bound** (network, file, DB) | GIL is released during I/O — threads work fine | Use `threading` or `asyncio` |

```python
# CPU-bound → use multiprocessing
from multiprocessing import Pool

def heavy_computation(n):
    return sum(i * i for i in range(n))

with Pool(4) as pool:
    results = pool.map(heavy_computation, [10**7] * 4)

# I/O-bound → threading works fine
import threading
import requests

def fetch(url):
    return requests.get(url)

threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
```

**Interview tip:** The GIL is a CPython implementation detail, not a Python language feature. Other implementations like Jython (Java) and IronPython (.NET) don't have a GIL. Python 3.13+ introduces an experimental "free-threaded" mode (`--disable-gil`) to remove this limitation.

---

## Python's Memory Management
---
Python manages memory automatically through a combination of:

### 1. Reference Counting
Every object has a counter tracking how many references point to it. When the count drops to `0`, the memory is freed immediately.

```python
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (one for 'a', one for getrefcount's argument)

b = a
print(sys.getrefcount(a))  # 3 (a, b, and getrefcount's argument)

del b
print(sys.getrefcount(a))  # 2 again
```

### 2. Garbage Collection (for circular references)
Reference counting can't handle circular references:

```python
class Node:
    def __init__(self):
        self.ref = None

# Circular reference
a = Node()
b = Node()
a.ref = b
b.ref = a

del a
del b
# Reference counts never reach 0!
# Python's garbage collector (gc module) detects and cleans these
```

```python
import gc

# Force garbage collection
gc.collect()

# Check garbage collector stats
print(gc.get_stats())
```

### 3. Memory Pools (PyMalloc)
Python uses a custom allocator for small objects (≤ 512 bytes):

```
OS Memory
  └── Python Memory Allocator (pymalloc)
        └── Arenas (256 KB chunks)
              └── Pools (4 KB, grouped by object size)
                    └── Blocks (individual objects)
```

This avoids calling the OS for every small allocation, making it much faster.

**Interview tip:** Python never returns memory to the OS for small objects. Even after `del`, the memory stays in Python's internal pool for reuse. Only large objects (> 512 bytes) are allocated/freed via the OS.

---

## List Comprehensions vs Generator Expressions
---

**List comprehension** — creates the entire list in memory at once:
```python
squares = [x**2 for x in range(1_000_000)]  # 8 MB in memory
```

**Generator expression** — produces items one at a time (lazy evaluation):
```python
squares = (x**2 for x in range(1_000_000))  # ~120 bytes (virtually nothing)
```

| Feature | List Comprehension `[]` | Generator Expression `()` |
| --- | --- | --- |
| Returns | `list` | `generator` object |
| Memory | Stores all items at once | Generates items on demand |
| Speed (single pass) | Slightly faster | Slightly slower per item |
| Reusable | Yes (iterate multiple times) | No (exhausted after one pass) |
| Supports indexing | Yes (`squares[5]`) | No |
| Best for | Small/medium data, need random access | Large data, single-pass processing |

```python
# Generator is exhausted after one use
gen = (x for x in range(5))
print(list(gen))  # [0, 1, 2, 3, 4]
print(list(gen))  # [] — already exhausted!
```

**When to use generators:**
- Processing large files line-by-line
- Streaming data from APIs
- Chaining transformations on large datasets
- Anywhere you need only one pass through data

```python
# Memory-efficient file processing
def read_large_file(path):
    with open(path) as f:
        for line in f:  # file objects are already generators!
            yield line.strip()

# Chain generators — almost zero memory
lines = read_large_file("huge_log.txt")
errors = (line for line in lines if "ERROR" in line)
first_10_errors = list(zip(range(10), errors))
```

---

## What are Python's Dunder (Magic) Methods?
---
Dunder methods (double underscore methods) are special methods that Python calls behind the scenes. They let you define how objects behave with built-in operations.

**Common dunder methods:**

| Method | Triggered by | Example |
| --- | --- | --- |
| `__init__` | Object creation | `obj = MyClass()` |
| `__str__` | `str()`, `print()` | `print(obj)` |
| `__repr__` | `repr()`, debugger | `repr(obj)` |
| `__len__` | `len()` | `len(obj)` |
| `__getitem__` | Indexing `[]` | `obj[0]` |
| `__setitem__` | Index assignment | `obj[0] = value` |
| `__contains__` | `in` operator | `item in obj` |
| `__eq__` | `==` | `obj1 == obj2` |
| `__lt__` | `<` | `obj1 < obj2` |
| `__add__` | `+` | `obj1 + obj2` |
| `__iter__` | `for` loop | `for x in obj` |
| `__call__` | Calling like function | `obj()` |
| `__enter__`/`__exit__` | `with` statement | `with obj as x:` |
| `__hash__` | `hash()`, dict keys | `{obj: value}` |

**Example — custom class with dunder methods:**
```python
class Money:
    def __init__(self, amount, currency="USD"):
        self.amount = amount
        self.currency = currency
    
    def __repr__(self):
        return f"Money({self.amount}, '{self.currency}')"
    
    def __str__(self):
        return f"{self.currency} {self.amount:.2f}"
    
    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)
    
    def __eq__(self, other):
        return self.amount == other.amount and self.currency == other.currency
    
    def __lt__(self, other):
        if self.currency != other.currency:
            raise ValueError("Cannot compare different currencies")
        return self.amount < other.amount

price1 = Money(10.50)
price2 = Money(20.00)

print(price1)              # USD 10.50
print(repr(price1))        # Money(10.5, 'USD')
print(price1 + price2)     # USD 30.50
print(price1 == price2)    # False
print(price1 < price2)     # True
```

**`__str__` vs `__repr__`:**
- `__str__`: Human-readable, used by `print()` and `str()`
- `__repr__`: Unambiguous, used by debugger, `repr()`, and as fallback for `__str__`
- Rule of thumb: `__repr__` should ideally return a string that could recreate the object

---

## What is the difference between `==` and `is`?
---

| Operator | Checks | Calls |
| --- | --- | --- |
| `==` | **Value equality** — do they have the same value? | `__eq__()` |
| `is` | **Identity** — are they the exact same object in memory? | Compares `id()` |

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True  — same value
print(a is b)   # False — different objects
print(a is c)   # True  — same object (c points to a)
```

**When to use `is`:**
- Comparing to `None`: always use `if x is None` (never `if x == None`)
- Comparing to singletons: `True`, `False`, `None`

**Python's integer caching gotcha:**
```python
# Python caches integers from -5 to 256
a = 256
b = 256
print(a is b)   # True — cached

a = 257
b = 257
print(a is b)   # False — not cached (may vary by implementation)
```

---

## What are Closures in Python?
---
A **closure** is a function that remembers variables from its enclosing scope, even after the outer function has finished executing.

**Three conditions for a closure:**
1. There must be a nested function (function inside a function)
2. The nested function must reference a variable from the enclosing function
3. The enclosing function must return the nested function

```python
def multiplier(factor):
    def multiply(number):     # nested function
        return number * factor  # references 'factor' from enclosing scope
    return multiply             # returns the nested function

double = multiplier(2)
triple = multiplier(3)

print(double(5))   # 10
print(triple(5))   # 15
print(double(10))  # 20

# 'factor' still lives inside the closure
print(double.__closure__[0].cell_contents)  # 2
```

**Practical example — counter:**
```python
def make_counter():
    count = 0
    def counter():
        nonlocal count   # needed to modify enclosing variable
        count += 1
        return count
    return counter

c = make_counter()
print(c())  # 1
print(c())  # 2
print(c())  # 3
```

**Common closure pitfall — late binding:**
```python
# BUG: all functions capture the same variable 'i'
functions = []
for i in range(5):
    functions.append(lambda: i)

print([f() for f in functions])  # [4, 4, 4, 4, 4] — all 4!

# FIX: use default argument to capture current value
functions = []
for i in range(5):
    functions.append(lambda i=i: i)  # default arg captures current 'i'

print([f() for f in functions])  # [0, 1, 2, 3, 4]
```

**Interview tip:** Closures are the foundation of decorators in Python. A decorator is essentially a closure that wraps a function.

---

## What is the difference between `@staticmethod` and `@classmethod`?
---

| | `@staticmethod` | `@classmethod` | Regular method |
| --- | --- | --- | --- |
| First argument | None | `cls` (the class) | `self` (the instance) |
| Can access instance? | No | No | Yes |
| Can access class? | Not directly | Yes (via `cls`) | Yes (via `self.__class__`) |
| Called on | Class or instance | Class or instance | Instance only |
| Use case | Utility function grouped in class | Factory methods, alternative constructors | Instance-specific behavior |

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day
    
    def __repr__(self):
        return f"Date({self.year}, {self.month}, {self.day})"
    
    # Regular method — needs an instance
    def is_leap_year(self):
        y = self.year
        return y % 4 == 0 and (y % 100 != 0 or y % 400 == 0)
    
    # Class method — alternative constructor
    @classmethod
    def from_string(cls, date_string):
        year, month, day = map(int, date_string.split("-"))
        return cls(year, month, day)  # creates a new instance
    
    # Static method — utility, no access to class or instance
    @staticmethod
    def is_valid_date(date_string):
        try:
            year, month, day = map(int, date_string.split("-"))
            return 1 <= month <= 12 and 1 <= day <= 31
        except ValueError:
            return False

# Usage
d = Date.from_string("2024-03-15")   # classmethod as factory
print(d)                               # Date(2024, 3, 15)
print(d.is_leap_year())                # True
print(Date.is_valid_date("2024-13-01"))  # False
```

---

## What is the `walrus operator` (`:=`)?
---
Introduced in **Python 3.8**, the walrus operator (`:=`) assigns a value to a variable **as part of an expression**.

```python
# Without walrus operator
data = input("Enter: ")
while data != "quit":
    print(f"You said: {data}")
    data = input("Enter: ")

# With walrus operator — cleaner
while (data := input("Enter: ")) != "quit":
    print(f"You said: {data}")
```

**Useful in list comprehensions:**
```python
# Without walrus — calls expensive_function() twice
results = [expensive_function(x) for x in data if expensive_function(x) > threshold]

# With walrus — calls once, saves result
results = [y for x in data if (y := expensive_function(x)) > threshold]
```

**Useful in `if` statements:**
```python
import re

text = "My phone number is 123-456-7890"
if (match := re.search(r'\d{3}-\d{3}-\d{4}', text)):
    print(f"Found: {match.group()}")  # Found: 123-456-7890
```

---

## References
1. [Python Official Documentation](https://docs.python.org/3/)
2. [Python Data Model (Dunder Methods)](https://docs.python.org/3/reference/datamodel.html)
3. [Python Memory Management](https://docs.python.org/3/c-api/memory.html)
4. [Collections — defaultdict](https://docs.python.org/3/library/collections.html#collections.defaultdict)
5. [Python GIL — Real Python](https://realpython.com/python-gil/)
6. [Functional Programming HOWTO](https://docs.python.org/3/howto/functional.html)