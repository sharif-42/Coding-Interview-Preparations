## Iterators and Generators

### What is an Iterator in Python?
---
An iterator in Python is an object used to traverse through all the elements of a collection (like lists, tuples or dictionaries) one element at a time. It follows the iterator protocol, which involves two key methods:

- `__iter__()`: Returns the iterator object itself.
- `__next__()`: Returns the next value from the sequence. Raises `StopIteration` when the sequence ends or no items are left.

**Iterator Protocol — Visual Flow:**
```
┌────────────────────────────────────────────────────────┐
│              ITERATOR PROTOCOL                            │
│                                                          │
│   Iterable              Iterator                         │
│   (list, str, dict)     (the actual worker)               │
│                                                          │
│   my_list = [1, 2, 3]                                    │
│       │                                                  │
│       │  iter(my_list)                                   │
│       ▼                                                  │
│   ┌──────────────┐                                       │
│   │  Iterator    │─── next() ──► 1                      │
│   │  object      │─── next() ──► 2                      │
│   │  (has state) │─── next() ──► 3                      │
│   │              │─── next() ──► StopIteration ⚠️         │
│   └──────────────┘                                       │
│                                                          │
│   Iterable: has __iter__() → returns an Iterator          │
│   Iterator: has __iter__() + __next__()                   │
│   ∴ Every Iterator is an Iterable, but not vice versa     │
└────────────────────────────────────────────────────────┘
```

**How a `for` loop actually works:**
```python
# This:
for item in [1, 2, 3]:
    print(item)

# Is equivalent to:
iterator = iter([1, 2, 3])   # calls __iter__()
while True:
    try:
        item = next(iterator)  # calls __next__()
        print(item)
    except StopIteration:
        break
```

**Why do we need iterators?** Here are some key benefits:
- Lazy Evaluation: Processes items only when needed, saving memory.
- Generator Integration: Pairs well with generators and functional tools.
- Stateful Traversal: Keeps track of where it left off.
- Uniform Looping: Same for loop works for lists, strings and more.
- Composable Logic: Easily build complex pipelines using tools like itertools.

**Built-in Iterator Example**
Let’s start with a simple example using a string. We will convert it into an iterator and fetch characters one by one:
```python
s = 'shariful'
iterator = iter(s)

print(next(iterator))
print(next(iterator))
print(next(iterator))
print(next(iterator))

# Sample output
s
h
a
r
```
**Explanation:**
- s is an iterable (string).
- iter(s) creates an iterator.
- next(iterator) retrieves characters one by one.

**Creating a Custom Iterator**

Creating a custom iterator in Python involves defining a class that implements the __iter__() and __next__() methods according to the Python iterator protocol.

Steps to follow:

- Define the Class: Start by defining a class that will act as the iterator.
- Initialize Attributes: In the __init__() method of the class, initialize any required attributes that will be used throughout the iteration process.
- Implement __iter__(): This method should return the iterator object itself. This is usually as simple as returning self.
- Implement __next__(): This method should provide the next item in the sequence each time it's called.

 Python uses iterators behind the scenes in:
- for loops
- List comprehensions
- Generators
- Most built-in data types (list, tuple, dict, set, etc.)

Below is an example of a custom class called EvenNumbers, which iterates through even numbers starting from 2:
```python
class EvenNumbers:
    def __iter__(self):
        self.n = 2  # Start from the first even number
        return self

    def __next__(self):
        x = self.n
        self.n += 2  # Increment by 2 to get the next even number
        return x

# Create an instance of EvenNumbers
even = EvenNumbers()
it = iter(even)

# Print the first five even numbers
print(next(it))  
print(next(it)) 
print(next(it))  
print(next(it)) 
print(next(it))
```
- Initialization: The __iter__() method initializes the iterator at 2, the first even number.
- Iteration: The __next__() method retrieves the current number and then increases it by 2, ensuring the next call returns the subsequent even number.
- Usage: We create an instance of EvenNumbers, turn it into an iterator and then use the next() function to fetch even numbers one at a time.

**Another example**

```python
class MyIterator:
    def __init__(self, n):
        self.n = n
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current < self.n:
            value = self.current
            self.current += 1
            return value
        else:
            raise StopIteration

# Example usage
my_iter = MyIterator(5)
for value in my_iter:
    print(value)  

Output: 
0
1
2
3
4
```
> Iterable: Something you can loop over (list, tuple, etc.)
>
> Iterator: The actual object that produces values one-by-one

**Why Iterators Matter?** 
- Save memory (no need to load all items)
- Useful for large data processing
- Foundation of generators (yield)
- Allow custom iteration behavior

**Iterable vs Iterator — Quick Reference:**

| | Iterable | Iterator |
| --- | --- | --- |
| Has `__iter__()` | ✓ | ✓ |
| Has `__next__()` | ✗ | ✓ |
| Can use in `for` loop | ✓ | ✓ |
| Can use `next()` on it | ✗ (need `iter()` first) | ✓ |
| Examples | list, tuple, str, dict, set | `iter([1,2,3])`, generators, file objects |
| Reusable | ✓ (create new iterator each time) | ✗ (exhausted after one pass) |

```python
# Iterable vs Iterator
my_list = [1, 2, 3]        # Iterable (has __iter__, no __next__)
my_iter = iter(my_list)     # Iterator (has __iter__ AND __next__)

print(hasattr(my_list, '__next__'))  # False
print(hasattr(my_iter, '__next__'))  # True

# Iterables can create multiple iterators
iter1 = iter(my_list)
iter2 = iter(my_list)
print(next(iter1))  # 1
print(next(iter2))  # 1  ← independent!
```


### What is an Generator in Python?
---
A generator function is a special type of function that returns an iterator object. Instead of using return to send back a single value, generator functions use yield to produce a series of results over time. This allows the function to generate values and pause its execution after each yield, maintaining its state between iterations.

A generator in Python is a special type of iterator that allows you to produce a sequence of values lazily — meaning one value at a time, only when needed.

Generators make it easy and memory-efficient to work with large datasets or streams of data.

**A generator is:**
- A function that uses `yield` instead of `return`
- Automatically creates an iterator (implements `__iter__` and `__next__` for you)
- Produces values on the fly instead of storing them in memory

**Generator Lifecycle Diagram:**
```
┌────────────────────────────────────────────────────────┐
│              GENERATOR LIFECYCLE                          │
│                                                          │
│  def my_gen():        gen = my_gen()                      │
│      yield 1          (returns generator object)          │
│      yield 2                                              │
│      yield 3                                              │
│                                                          │
│  ┌──────────┐   next()   ┌──────────────────────┐    │
│  │ CREATED  │────────►│  RUNNING               │    │
│  └──────────┘           │  executes until yield  │    │
│                         └─────────┬────────────┘    │
│                                   │ yield value          │
│                                   ▼                      │
│                         ┌──────────────────────┐    │
│                         │  SUSPENDED             │    │
│                    ┌────│  (paused, state saved)  │    │
│             next() │    └──────────────────────┘    │
│                    │              │ (no more yields)     │
│                    ▼              ▼                      │
│               back to      ┌─────────────────┐       │
│               RUNNING      │  EXHAUSTED       │       │
│                            │  StopIteration   │       │
│                            └─────────────────┘       │
└────────────────────────────────────────────────────────┘
```

**Why Generators Are Useful?**
- Faster for large data: Only computes next item when requested
- Clean, simple syntax: Uses yield
- Works like an iterator: Can use next() or a for loop
- Memory Efficient : Handle large or infinite data without loading everything into memory.
- No List Overhead : Yield items one by one, avoiding full list creation.
- Lazy Evaluation : Compute values only when needed, improving performance.
- Support Infinite Sequences : Ideal for generating unbounded data like Fibonacci series.
- Pipeline Processing : Chain generators to process data in stages efficiently.

```python
def count_up_to(n):
    current = 1
    while current <= n:
        yield current
        current += 1

# Usage
for number in count_up_to(5):
    print(number)


gen = count_up_to(3)

print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # 3
print(next(gen))  # StopIteration
```

**How Generators Work Internally**

When a generator function is called:

- It does not run immediately.
- It returns a generator object.
- Each call to `next()` resumes execution until it hits `yield`, then pauses.
- The function's local variables and execution state are preserved between calls.

```
                      Generator Function Execution

call my_gen()  ──►  returns generator object (no code runs yet)
                       │
next(gen)      ──►  runs code ──► hits yield 1 ──► PAUSES ─► returns 1
                       │
next(gen)      ──►  resumes   ──► hits yield 2 ──► PAUSES ─► returns 2
                       │
next(gen)      ──►  resumes   ──► hits yield 3 ──► PAUSES ─► returns 3
                       │
next(gen)      ──►  resumes   ──► function ends ─► raises StopIteration
```

**Generator Expression (one-liner):**
```python
# Generator expression — like list comprehension but with ()
squares_gen = (x**2 for x in range(1_000_000))  # ~120 bytes
squares_list = [x**2 for x in range(1_000_000)] # ~8 MB

import sys
print(sys.getsizeof(squares_gen))   # 200 (tiny!)
print(sys.getsizeof(squares_list))  # 8,448,728 (8 MB!)
```

**Reading Large Files Lazily**

When you have a huge file, reading it all at once can consume a lot of memory. Generators let you process it line by line.

```python
def read_large_file(file_path):
    with open(file_path, "r") as f:
        for line in f:
            yield line.strip()  # yield one line at a time

# Usage
for line in read_large_file("bigfile.txt"):
    process(line)  # do something with each line
```

**Benefits:**
- Only one line is in memory at a time
- Perfect for log processing, ETL pipelines, or streaming data

**Streaming API Responses**

When dealing with APIs that return paginated or streaming data, a generator can yield data page by

```python
import requests

def fetch_users(api_url):
    page = 1
    while True:
        response = requests.get(f"{api_url}?page={page}")
        data = response.json()
        if not data["results"]:
            break
        for user in data["results"]:
            yield user
        page += 1

# Usage
for user in fetch_users("https://example.com/api/users"):
    print(user["name"])
```

**Benefits:**

- Avoid loading all results in memory
- Processes each item immediately
- Handles infinite or very large streams efficiently

**Iterating Over Database Rows**

When querying large tables, fetching all rows at once can crash your app. Generators let you fetch in batches.

```python
import sqlite3

def get_users_in_batches(batch_size=100):
    conn = sqlite3.connect("mydb.sqlite")
    cursor = conn.cursor()
    offset = 0
    while True:
        cursor.execute("SELECT id, name FROM users LIMIT ? OFFSET ?", (batch_size, offset))
        rows = cursor.fetchall()
        if not rows:
            break
        for row in rows:
            yield row
        offset += batch_size
    conn.close()

# Usage
for user_id, name in get_users_in_batches():
    print(user_id, name)
```
**Benefits:**

- Memory-efficient iteration over large tables
- Works for reporting, ETL, or background tasks

### Why are generators preferred for handling large datasets?
---

Generators are preferred for large datasets because they use lazy evaluation (they produce values one at a time, only when requested by `next()`). They maintain their state between calls but do not store the entire sequence in memory, resulting in significant memory savings.

**Memory Efficiency**

- Generators produce one item at a time instead of storing the entire dataset in memory.
- Lists or other collections load all items at once, which can be huge and lead to MemoryError for large datasets.

Example:

```python
# List comprehension (loads all at once)
numbers = [x*x for x in range(1_000_000_000)]  # might crash

# Generator (lazy evaluation)
numbers = (x*x for x in range(1_000_000_000))  # uses very little memory
```
Only the current value exists in memory, not the whole dataset.

**Lazy Evaluation**

Generators compute values only when needed (on-demand).

- Useful for streaming data, APIs, or file processing.
- Example: Processing a huge log file

```python
def read_log(file_path):
    with open(file_path) as f:
        for line in f:
            yield line.strip()  # read line by line

for line in read_log("big_log.txt"):
    process(line)
```
The file is processed line by line, never fully loaded into memory.


**Performance (Faster for Sequential Processing)**

- Generators avoid the overhead of creating and storing huge lists.
- Execution starts immediately and produces first results quickly.
- Ideal for pipelines where intermediate results are processed immediately.

Example: Chained generator pipeline

```python
def numbers(n):
    for i in range(n):
        yield i

def squares(nums):
    for x in nums:
        yield x*x

for sq in squares(numbers(1_000_000)):
    if sq > 100_000:
        break
```
Each value is generated and processed on the fly, saving both memory and time.

### Generator Pipeline Pattern
---

Generators can be chained together to create data processing pipelines — like Unix pipes.

```
┌────────────────────────────────────────────────────────┐
│              GENERATOR PIPELINE                          │
│                                                          │
│  Data Source     Stage 1       Stage 2       Consumer    │
│  ┌────────┐   ┌────────┐  ┌────────┐  ┌────────┐  │
│  │ read   │──►│ filter │─►│ trans- │─►│ output │  │
│  │ lines  │   │ errors │  │ form   │  │ to DB  │  │
│  └────────┘   └────────┘  └────────┘  └────────┘  │
│    yield ──►    yield ──►   yield ──►   consume    │
│                                                          │
│  Only ONE item flows through the entire pipeline at a     │
│  time ─ virtually zero memory regardless of data size!    │
└────────────────────────────────────────────────────────┘
```

**Practical pipeline example — processing a log file:**
```python
def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line.strip()

def filter_errors(lines):
    for line in lines:
        if "ERROR" in line:
            yield line

def extract_message(lines):
    for line in lines:
        # "2024-01-15 ERROR: Connection timeout" → "Connection timeout"
        yield line.split("ERROR:")[-1].strip()

# Chain the pipeline — each item flows through one at a time
lines = read_lines("server.log")          # Stage 1: read
errors = filter_errors(lines)              # Stage 2: filter
messages = extract_message(errors)         # Stage 3: transform

for msg in messages:                       # Consumer: process
    print(msg)

# Memory usage: ~1 line at a time regardless of file size!
```

### `yield from` — Delegating to Sub-generators
---

`yield from` delegates iteration to another iterable or generator, simplifying nested generators:

```python
# Without yield from
def flatten(nested_list):
    for sublist in nested_list:
        for item in sublist:
            yield item

# With yield from (cleaner)
def flatten(nested_list):
    for sublist in nested_list:
        yield from sublist  # delegates to sublist's iterator

print(list(flatten([[1, 2], [3, 4], [5]])))  # [1, 2, 3, 4, 5]
```

**Recursive generator with `yield from`:**
```python
def flatten_deep(data):
    """Flatten arbitrarily nested lists."""
    for item in data:
        if isinstance(item, list):
            yield from flatten_deep(item)  # recurse
        else:
            yield item

print(list(flatten_deep([1, [2, [3, [4]], 5], 6])))
# [1, 2, 3, 4, 5, 6]
```

### `send()`, `throw()`, and `close()` — Advanced Generator Methods
---

Generators are not just producers — they can also **receive** values.

```python
def accumulator():
    total = 0
    while True:
        value = yield total   # yield current total, receive next value
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)           # Prime the generator (advance to first yield)→ 0
gen.send(10)        # Send 10 → total=10, yields 10
gen.send(20)        # Send 20 → total=30, yields 30
gen.send(5)         # Send 5  → total=35, yields 35
```

```
send() flow:

  Caller                    Generator
    │                          │
    │── next(gen) ──────────►│  (prime → run to first yield)
    │                   0 ◄──│  yield total
    │                          │  PAUSED
    │── gen.send(10) ──────►│  value = 10, total = 10
    │                  10 ◄──│  yield total
    │                          │  PAUSED
    │── gen.send(20) ──────►│  value = 20, total = 30
    │                  30 ◄──│  yield total
```

| Method | Purpose |
| --- | --- |
| `next(gen)` | Advance to next yield, get value |
| `gen.send(value)` | Send a value INTO the generator (resumes it) |
| `gen.throw(ExceptionType)` | Raise an exception inside the generator |
| `gen.close()` | Terminate the generator (raises `GeneratorExit`) |

### Useful `itertools` Functions
---

The `itertools` module provides powerful iterator-building blocks:

```python
import itertools

# chain — concatenate iterables
list(itertools.chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]

# islice — slice an iterator (like list slicing for generators)
gen = (x**2 for x in range(1000))
list(itertools.islice(gen, 5))  # [0, 1, 4, 9, 16] (first 5 only)

# count — infinite counter
for i in itertools.islice(itertools.count(10, 2), 5):
    print(i)  # 10, 12, 14, 16, 18

# groupby — group consecutive items
data = [('A', 1), ('A', 2), ('B', 3), ('B', 4)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# A [('A', 1), ('A', 2)]
# B [('B', 3), ('B', 4)]

# product — cartesian product
list(itertools.product('AB', '12'))  # [('A','1'), ('A','2'), ('B','1'), ('B','2')]

# combinations & permutations
list(itertools.combinations('ABC', 2))  # [('A','B'), ('A','C'), ('B','C')]
list(itertools.permutations('AB', 2))   # [('A','B'), ('B','A')]
```

### Yield vs Return
---
**Yield**: is used in generator functions to provide a sequence of values over time. When yield is executed, it pauses the function, returns the current value and retains the state of the function. This allows the function to continue from same point when called again, making it ideal for generating large or complex sequences efficiently.

**Return**: is used to exit a function and return a final value. Once return is executed, function is terminated immediately and no state is retained. This is suitable for cases where a single result is needed from a function.

**Side-by-side comparison:**

| Feature | `return` | `yield` |
| --- | --- | --- |
| Terminates function? | Yes | No (pauses) |
| State preserved? | No | Yes |
| Returns | Single value | Generator object (produces many values) |
| Memory | All at once | One value at a time |
| Can be iterated? | No (unless returns a list) | Yes (is an iterator) |
| Resumable? | No | Yes (continues from where it paused) |

```python
# return — creates entire list in memory
def get_squares_return(n):
    result = []
    for i in range(n):
        result.append(i ** 2)
    return result              # all values computed and stored

# yield — produces values one at a time
def get_squares_yield(n):
    for i in range(n):
        yield i ** 2           # one value at a time, state preserved

# Memory comparison for n = 10,000,000:
# return version: ~80 MB (stores all 10M integers)
# yield version:  ~120 bytes (only 1 integer at a time)
```

### Infinite Generators
---

Generators can produce **infinite sequences** since they only compute one value at a time:

```python
def fibonacci():
    a, b = 0, 1
    while True:      # infinite loop — but safe because it yields!
        yield a
        a, b = b, a + b

# Take only what you need
import itertools
first_10 = list(itertools.islice(fibonacci(), 10))
print(first_10)  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Find first fibonacci number > 1000
for fib in fibonacci():
    if fib > 1000:
        print(fib)  # 1597
        break
```

---

### Common Interview Questions
---

**Q: What is the difference between an iterable and an iterator?**
An **iterable** is any object that has `__iter__()` (lists, strings, dicts). An **iterator** is an object that also has `__next__()` and maintains state. You get an iterator by calling `iter()` on an iterable. Every iterator is an iterable, but not every iterable is an iterator.

**Q: What happens when you call a generator function?**
The function body does NOT execute. Instead, it returns a **generator object**. The code inside only runs when you call `next()` on the generator or iterate over it with a `for` loop.

**Q: Can a generator be restarted after it is exhausted?**
No. Once a generator raises `StopIteration`, it cannot be restarted. You must create a new generator object by calling the generator function again.

**Q: What is the difference between a generator function and a generator expression?**
- Generator function: uses `def` and `yield` — `def my_gen(): yield 1`
- Generator expression: uses parentheses — `(x for x in range(10))`
Both produce generator objects. Use expressions for simple cases, functions for complex logic.

**Q: What is `yield from` and when would you use it?**
`yield from` delegates iteration to another iterable. Use it when you want to yield all values from a sub-generator or iterable without writing a `for` loop. It's especially useful in recursive generators (like flattening nested structures).

**Q: How is a generator different from a list comprehension?**

```
List:       [x**2 for x in range(1M)]  → 8 MB in memory, all at once
Generator:  (x**2 for x in range(1M))  → ~120 bytes, one at a time
```
Use a list when you need random access or multiple iterations. Use a generator for single-pass processing, large datasets, or when memory is a concern.

**Q: What does `gen.send(value)` do?**
`send()` resumes the generator and sends a value into it. The sent value becomes the result of the `yield` expression inside the generator. You must call `next(gen)` first to "prime" the generator (advance it to the first `yield`).

**Q: When should you use iterators/generators vs lists?**

| Use Lists when... | Use Generators when... |
| --- | --- |
| You need random access (`items[5]`) | You're processing data sequentially |
| You need to iterate multiple times | One pass through the data is enough |
| Dataset fits comfortably in memory | Dataset is large or infinite |
| You need `len()`, slicing, etc. | Memory efficiency is critical |

---

### References
1. [Python Iterator Protocol — Official Docs](https://docs.python.org/3/library/stdtypes.html#iterator-types)
2. [Generators — Official Docs](https://docs.python.org/3/howto/functional.html#generators)
3. [itertools Module](https://docs.python.org/3/library/itertools.html)
4. [PEP 255 — Simple Generators](https://peps.python.org/pep-0255/)
5. [PEP 380 — Syntax for Delegating to a Subgenerator (yield from)](https://peps.python.org/pep-0380/)