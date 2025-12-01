## Decorators in Python

In Python, decorators are flexible way to modify or extend behavior of functions or methods, without changing their actual code.

A decorator is essentially a function that takes another function as an argument and returns a new function with enhanced functionality.
Decorators are often used in scenarios such as logging, authentication and memorization, allowing us to add additional functionality to existing functions or methods in a clean, reusable way.

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
- Logging: Track function calls (e.g., @logger).
- Authentication: Restrict access in web apps (e.g., Flask/Django).
- Rate Limiting: Control API usage per user.
- Caching: Store results using functools.lru_cache.
- Retry Logic: Automatically retry failed network calls.