## Decorators in Python

In Python, decorators are a flexible way to modify or extend the behavior of functions or methods, without changing their actual code.

A decorator is essentially a function that takes another function as an argument and returns a new function with enhanced functionality.
Decorators are often used in scenarios such as logging, authentication and memoization, allowing us to add additional functionality to existing functions or methods in a clean, reusable way.

### How Decorators Work — Visual Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    DECORATOR PATTERN                        │
│                                                             │
│   @decorator                                                │
│   def greet():          is equivalent to:                   │
│       ...               greet = decorator(greet)            │
│                                                             │
│   ┌──────────────┐      ┌──────────────┐                    │
│   │   Original   │      │  decorator() │                    │
│   │   greet()    │─────►│  takes func  │                    │
│   └──────────────┘      │  as argument │                    │
│                         └──────┬───────┘                    │
│                                │                            │
│                                ▼                            │
│                         ┌──────────────┐                    │
│                         │  wrapper()   │                    │
│                         │  ┌────────┐  │                    │
│                         │  │ before │  │                    │
│                         │  │ func() │  │ ◄── returned as   │
│                         │  │ after  │  │     new "greet"    │
│                         │  └────────┘  │                    │
│                         └──────────────┘                    │
│                                                             │
│   When you call greet(), you're actually calling wrapper()  │
└─────────────────────────────────────────────────────────────┘
```

Example: Let's see an example in which a simple Python decorator adds behavior before and after a function call.

```python
def decorator(func):
    def wrapper():
        print("Before calling the function.")
        func()
        print("After calling the function.")
    return wrapper

@decorator # Applying the decorator to a function
def greet():
    print("Hello, World!")
greet()

# Output
Before calling the function.
Hello, World!
After calling the function.
```
**Explanation**
- decorator takes the greet function as an argument.
- It returns a new function (wrapper) that first prints a message, calls greet() and then prints another message.
- @decorator syntax is a shorthand for greet = decorator(greet).

**Decorator with Parameters**

Decorators often need to work with functions that have arguments. We use `*args` and `**kwargs` so our wrapper can accept any number of arguments.

Example: Let's see an example of a decorator that adds functionality before and after a function call.

```python
def decorator_name(func):
    def wrapper(*args, **kwargs):
        print("Before execution")
        result = func(*args, **kwargs)
        print("After execution")
        return result
    return wrapper

@decorator_name
def add(a, b):
    return a + b

print(add(5, 3))

# Output
Before execution
After execution
8
```
- decorator_name(func): This is the decorator function. It takes another function (func) as input.
- **wrapper(*args, kwargs): A nested function that wraps func. *args collects positional arguments, **kwargs collects keyword arguments, so wrapper works with any function.
- @decorator_name: Syntax sugar for add = decorator_name(add).

### The `functools.wraps` Problem
---

When you wrap a function with a decorator, the original function's metadata (name, docstring) is lost:

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def say_hello():
    """This function says hello."""
    print("Hello!")

print(say_hello.__name__)  # 'wrapper' ← WRONG! Should be 'say_hello'
print(say_hello.__doc__)   # None ← WRONG! Lost the docstring
```

**Fix with `functools.wraps`:**
```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # ← preserves original function metadata
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def say_hello():
    """This function says hello."""
    print("Hello!")

print(say_hello.__name__)  # 'say_hello' ✓
print(say_hello.__doc__)   # 'This function says hello.' ✓
```

**Always use `@wraps(func)` in your decorators.** It preserves `__name__`, `__doc__`, `__module__`, and other attributes.

### Decorator with Arguments (Decorator Factory)
---

Sometimes you need to pass arguments to the decorator itself. This requires **three levels of nesting**:

```
┌────────────────────────────────────────────────┐
│  decorator_factory(arg)          ← Level 1     │
│  │                               (takes args)  │
│  └──► decorator(func)            ← Level 2     │
│       │                          (takes func)  │
│       └──► wrapper(*args, **kw)  ← Level 3     │
│                                  (replaces fn) │
└────────────────────────────────────────────────┘
```

```python
from functools import wraps

def repeat(n):                        # Level 1: takes decorator arguments
    def decorator(func):              # Level 2: takes the function
        @wraps(func)
        def wrapper(*args, **kwargs): # Level 3: replaces the function
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)         # repeat(3) returns decorator, which wraps say_hello
def say_hello():
    print("Hello!")

say_hello()
# Hello!
# Hello!
# Hello!
```

### Chaining (Stacking) Multiple Decorators
---

You can apply multiple decorators to a single function. They execute **bottom-up** (closest to the function first).

```python
@decorator_A
@decorator_B
@decorator_C
def my_function():
    pass

# Equivalent to:
# my_function = decorator_A(decorator_B(decorator_C(my_function)))
```

**Execution order diagram:**
```
    Call my_function()
           │
           ▼
  ┌─────────────────┐
  │  decorator_A    │  ← executes its "before" code FIRST
  │  ┌─────────────┐│
  │  │ decorator_B ││  ← executes its "before" code SECOND
  │  │ ┌─────────┐ ││
  │  │ │  dec_C  │ ││  ← executes its "before" code THIRD
  │  │ │ ┌─────┐ │ ││
  │  │ │ │func │ │ ││  ← original function runs
  │  │ │ └─────┘ │ ││
  │  │ │  dec_C  │ ││  ← executes its "after" code THIRD
  │  │ └─────────┘ ││
  │  │ decorator_B ││  ← executes its "after" code SECOND
  │  └─────────────┘│
  │  decorator_A    │  ← executes its "after" code FIRST
  └─────────────────┘
```

**Practical example:**
```python
from functools import wraps

def bold(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return f"<b>{func(*args, **kwargs)}</b>"
    return wrapper

def italic(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return f"<i>{func(*args, **kwargs)}</i>"
    return wrapper

@bold
@italic
def greet(name):
    return f"Hello, {name}"

print(greet("Alice"))  # <b><i>Hello, Alice</i></b>
# italic runs first (closest to function), then bold wraps the result
```

### Class-Based Decorators
---

You can use a class as a decorator by implementing `__call__`:

```python
from functools import wraps

class CountCalls:
    """Decorator that counts how many times a function is called."""
    def __init__(self, func):
        wraps(func)(self)     # preserve metadata
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} has been called {self.count} times")
        return self.func(*args, **kwargs)

@CountCalls
def say_hello():
    print("Hello!")

say_hello()  # say_hello has been called 1 times → Hello!
say_hello()  # say_hello has been called 2 times → Hello!
say_hello()  # say_hello has been called 3 times → Hello!
```

**When to use class-based decorators:**
- When the decorator needs to maintain state (like the count above)
- When you want a more complex decorator with multiple methods

### Decorating Classes (not just functions)
---

Decorators can also be applied to classes to modify or enhance them:

```python
def add_repr(cls):
    """Decorator that adds a __repr__ method to a class."""
    def __repr__(self):
        attrs = ', '.join(f'{k}={v!r}' for k, v in self.__dict__.items())
        return f"{cls.__name__}({attrs})"
    cls.__repr__ = __repr__
    return cls

@add_repr
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

print(User("Alice", 30))  # User(name='Alice', age=30)
```

**Note:** Python's built-in `@dataclass` is essentially a class decorator that auto-generates `__init__`, `__repr__`, `__eq__`, etc.

### Functions as First-Class Objects
---
In Python, functions are `first-class objects`, meaning they can be treated like any other object (such as integers, strings or lists). This allows functions to be assigned to variables, passed as arguments, returned from other functions and stored in data structures enabling flexible programming patterns, including decorators.

Example: This code demonstrates all four properties of functions as first-class objects in Python.

```python
# Assigning a function to a variable
def greet(n):
    return f"Hello, {n}!"
say_hi = greet  # Assign the greet function to say_hi
print(say_hi("Alice")) 

# Passing a function as an argument
def apply(f, v):
    return f(v)
res = apply(say_hi, "Bob")
print(res) 

# Returning a function from another function
def make_mult(f):
    def mult(x):
        return x * f
    return mult
dbl = make_mult(2)
print(dbl(5))
```

**Explanation:**

- greet() is assigned to say_hi variable, which is used to print a greeting for "Alice".
- apply() takes a function and a value as arguments, applies function to value and returns the result.
- apply is demonstrated by passing say_hi and "Bob", printing a greeting for "Bob".
-   make_mult() creates a multiplier function based on a given factor.

**Role of First-Class Functions in Decorators**

- Decorators take a function as input to modify or enhance its behavior.
- They return a new function that wraps the original, adding behavior before or after execution.
- The original function is replaced by decorated function when assigned to same name.

### Higher Order Functions in Python
---
Higher Order Functions (HOFs) play an important role in functional programming and allow for writing more modular, reusable and readable code. A Higher-Order Function is a function that either:

- Takes another function as an argument
- Returns a function as a result

```python
def greet(func): # higher-order function 
    return func("Hello")

def uppercase(text): # function to be passed
    return text.upper()

print(greet(uppercase))
```

**Real-World Uses of Decorators**

```
┌──────────────────────────────────────────────────────────┐
│              REAL-WORLD DECORATOR USE CASES               │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐    │
│  │  Logging   │  │   Auth     │  │  Rate Limiting   │    │
│  │  @logger   │  │  @login    │  │  @rate_limit     │    │
│  │            │  │  _required │  │  (100/hour)      │    │
│  └────────────┘  └────────────┘  └──────────────────┘    │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐    │
│  │  Caching   │  │   Retry    │  │  Timing/Perf     │    │
│  │ @lru_cache │  │ @retry(3)  │  │ @timer           │    │
│  │            │  │            │  │                  │    │
│  └────────────┘  └────────────┘  └──────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

**1. Logging Decorator:**
```python
from functools import wraps
import logging

def log_calls(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        logging.info(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        logging.info(f"{func.__name__} returned {result}")
        return result
    return wrapper

@log_calls
def add(a, b):
    return a + b
```

**2. Timing Decorator:**
```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "done"

slow_function()  # slow_function took 1.0012s
```

**3. Retry Decorator (with arguments):**
```python
import time
from functools import wraps

def retry(max_attempts=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"Attempt {attempt} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            raise Exception(f"All {max_attempts} attempts failed")
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def call_external_api():
    # may fail due to network issues
    pass
```

**4. Caching with `functools.lru_cache`:**
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(100))  # Instant! Cached results
print(fibonacci.cache_info())  # CacheInfo(hits=98, misses=101, ...)
```

```
Without cache:  fibonacci(35) → ~4.5 seconds (exponential calls)
With @lru_cache: fibonacci(35) → ~0.00001 seconds (cached)

   fib(5)
   ├── fib(4) ★
   │   ├── fib(3) ★
   │   │   ├── fib(2) ★
   │   │   └── fib(1) ← cached
   │   └── fib(2) ← cached (already computed)
   └── fib(3) ← cached (already computed)

   ★ = computed once, then cached
```

**5. Authentication Decorator (Flask-style):**
```python
from functools import wraps

def login_required(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        if not current_user.is_authenticated:
            return redirect('/login')
        return func(*args, **kwargs)
    return wrapper

@app.route('/dashboard')
@login_required
def dashboard():
    return render_template('dashboard.html')
```

---

### Built-in Decorators in Python
---

| Decorator | Purpose | Example |
| --- | --- | --- |
| `@staticmethod` | Method that doesn't access instance or class | Utility functions inside a class |
| `@classmethod` | Method that receives the class as first arg | Factory methods, alternative constructors |
| `@property` | Makes a method behave like an attribute | Computed attributes with getters/setters |
| `@functools.wraps` | Preserves metadata of wrapped function | Inside every decorator |
| `@functools.lru_cache` | Caches function results | Expensive computations, recursive functions |
| `@functools.singledispatch` | Function overloading by argument type | Type-based dispatch |
| `@dataclasses.dataclass` | Auto-generates `__init__`, `__repr__`, etc. | Data holder classes |
| `@abc.abstractmethod` | Forces subclasses to implement a method | Abstract base classes |

**`@property` decorator example:**
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
print(c.radius)    # 5     ← accessed like attribute, but calls method
print(c.area)      # 78.54 ← computed property
c.radius = 10      # ← calls the setter
# c.radius = -1    # ← raises ValueError
```

---

### Common Interview Questions
---

**Q: What is a decorator in Python?**
A decorator is a function that takes another function as an argument and returns a new function that extends or modifies the original function's behavior without changing its code. The `@decorator` syntax is syntactic sugar for `func = decorator(func)`.

**Q: What is the difference between `@staticmethod`, `@classmethod`, and a regular method?**
- Regular method: receives `self` (instance) as first argument
- `@classmethod`: receives `cls` (class) as first argument — used for factory methods
- `@staticmethod`: receives no implicit first argument — used for utility functions grouped in a class

**Q: What does `@functools.wraps` do and why is it important?**
`@wraps(func)` preserves the original function's metadata (`__name__`, `__doc__`, `__module__`, etc.) on the wrapper function. Without it, debugging and introspection tools would see the wrapper's metadata instead of the original function's.

**Q: Can you chain multiple decorators? What's the execution order?**
Yes. Decorators are applied bottom-up (closest to the function first), but their "before" code executes top-down when the function is called. This is because each decorator wraps the result of the one below it.

**Q: What is the difference between a decorator and a decorator factory?**
- **Decorator**: takes a function → returns a wrapper. Usage: `@my_decorator`
- **Decorator factory**: takes arguments → returns a decorator. Usage: `@my_decorator(arg1, arg2)`. This requires an extra level of nesting.

**Q: How would you write a decorator that works on both functions and methods?**
Use `*args` and `**kwargs` in the wrapper. For methods, `self` is automatically passed as the first positional argument through `*args`.

**Q: What is `@lru_cache` and when would you use it?**
`@lru_cache` is a built-in decorator from `functools` that caches function results based on arguments. Use it for expensive computations with repeated inputs (e.g., recursive Fibonacci, API calls with the same parameters). The `maxsize` parameter limits cache size (LRU eviction).

---

### References
1. [Python Decorators — Official Docs](https://docs.python.org/3/glossary.html#term-decorator)
2. [functools — Higher-order functions](https://docs.python.org/3/library/functools.html)
3. [PEP 318 — Decorators for Functions and Methods](https://peps.python.org/pep-0318/)
4. [Primer on Python Decorators — Real Python](https://realpython.com/primer-on-python-decorators/)