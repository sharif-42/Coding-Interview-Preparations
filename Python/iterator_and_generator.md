## Iterators and Generators

### What is an Iterator in Python?
---
An iterator in Python is an object used to traverse through all the elements of a collection (like lists, tuples or dictionaries) one element at a time. It follows the iterator protocol, which involves two key methods:

- __iter__(): Returns the iterator object itself.
- __next__(): Returns the next value from the sequence. Raises StopIteration when the sequence ends or no items are left

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


### What is an Generator in Python?
---
A generator function is a special type of function that returns an iterator object. Instead of using return to send back a single value, generator functions use yield to produce a series of results over time. This allows the function to generate values and pause its execution after each yield, maintaining its state between iterations.

A generator in Python is a special type of iterator that allows you to produce a sequence of values lazily — meaning one value at a time, only when needed.

Generators make it easy and memory-efficient to work with large datasets or streams of data.

**A generator is:**
- A function that uses yield instead of return
- Automatically creates an iterator
- Produces values on the fly instead of storing them in memory

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
- Each call to next() resumes execution until it hits yield, then pauses.

```python
start → run until yield → pause → resume → yield → pause → ... → StopIteration
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

### Yield vs Return
---
**Yield**: is used in generator functions to provide a sequence of values over time. When yield is executed, it pauses the function, returns the current value and retains the state of the function. This allows the function to continue from same point when called again, making it ideal for generating large or complex sequences efficiently.

**Return**: is used to exit a function and return a final value. Once return is executed, function is terminated immediately and no state is retained. This is suitable for cases where a single result is needed from a function.