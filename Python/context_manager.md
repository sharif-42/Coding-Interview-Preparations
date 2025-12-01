## Context Managers and Generators


### What is a context manager, and how can you implement one using the contextlib module?
---

Context managers provide a neat way to automatically set up and clean up resources, ensuring they’re properly managed even if errors occur.

**Why Do We Need Context Managers?**

- Automatic cleanup: Frees resources like files or connections without manual effort.
- Avoid resource leaks: Ensures proper release, even if errors occur.
- Cleaner code: Replaces verbose try-finally blocks with with.
- Exception-safe: Always performs cleanup, even on exceptions.
- Custom control: Lets you manage any resource using __enter__ and __exit__.

**Built-in Context Manager for File Handling**

File operations are a common case in Python where proper resource management is crucial. The with statement provides a built-in context manager that ensures file is automatically closed once you're done with it, even if an error occurs.

Example: In this Example, we open a file using a context manager with statement and reads its content safely.
```python
with open("test.txt") as f:
    data = f.read()
```
This eliminates need to explicitly call close() and protects against resource leakage in case of unexpected failures.

**Creating Custom Context Manager Class **

A class-based context manager needs two methods:
- __enter__(): sets up the resource and returns it.
- __exit__(): cleans up the resource (e.g., closes a file).

The below example, shows how Python initializes the object, enters the context, runs the block and then exits while cleaning up.

```python
class ContextManager:
    def __init__(self):
        print('init method called')
        
    def __enter__(self):
        print('enter method called')
        return self
    
    def __exit__(self, exc_type, exc_value, exc_traceback):
        print('exit method called')

with ContextManager() as manager:
    print('with statement block')

# sample output
# init method called
# enter method called
# with statement block
# exit method called
```
Explanation:
- __init__() initializes the object.
- __enter__() runs at the start of the with block and returns the object.
- The block executes (print statement).
- __exit__() is called after the block ends to handle cleanup.

**File Management Using Context Manager**

Applying the concept of custom context managers, this example defines a FileManager class to handle opening, reading/writing and automatically closing a file.

```python
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
        
    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_value, exc_traceback):
        self.file.close()

with FileManager('test.txt', 'w') as f:
    f.write('Test')

print(f.closed)
```
- __enter__() opens the file and returns it.
- Inside the with block, you can use the file object.
- __exit__() ensures the file is closed automatically.
- print(f.closed) confirms the file is closed.

**Database Connection Management with Context Manager**

Let's create a simple database connection management system. The number of database connections that can be opened at a time is also limited(just like file descriptors). Therefore context managers are helpful in managing connections to the database as there could be chances that programmer may forget to close the connection.

```python
from pymongo import MongoClient

class MongoDBConnectionManager:
    def __init__(self, hostname, port):
        self.hostname = hostname
        self.port = port
        self.connection = None

    def __enter__(self):
        self.connection = MongoClient(self.hostname, self.port)
        return self.connection

    def __exit__(self, exc_type, exc_value, traceback):
        self.connection.close()

with MongoDBConnectionManager('localhost', 27017) as mongo:
    collection = mongo.SampleDb.test
    data = collection.find_one({'_id': 1})
    print(data.get('name'))
```
- __enter__() opens MongoDB connection.
- mongo inside with block is the client object.
- You can perform database operations safely.
- __exit__() closes the connection automatically.

**Create context manager using contextlib — Easier Way**

The contextlib module provides helper tools to create context managers without writing a class. The most common way is using the `@contextmanager` decorator.

```python
from contextlib import contextmanager
import time

@contextmanager
def timer():
    start_time = time.time()
    yield
    end_time = time.time()
    elapsed_time = end_time - start_time
    print(f"Elapsed time: {elapsed_time} seconds")

with timer():
    sum(range(10_000_000))
```

**Summary**

A context manager handles setup and teardown of resources using __enter__ and __exit__. You use it with the with statement to guarantee cleanup, even if errors occur.It can be implemented manually using a class or easily using the @contextmanager decorator from contextlib.


### Question: Explain a context manager while we are writting django queries?
---

<b>Answer</b>: In Django, whenever you execute queries—especially inside transaction.atomic()—Django uses its own built-in transaction context manager implemented in `django/db/transaction.py`

The main context manager used under the hood is Atomic, from django.db.transaction. When you write:
```python
from django.db import transaction

with transaction.atomic():
    User.objects.create(name="John")
    Profile.objects.create(user_id=1)
```
Django is using the Atomic context manager internally.

Internally it:
- Starts a DB transaction
- Executes the code block
- If everything succeeds → COMMIT
- If any exception occurs → ROLLBACK
- Supports nested transactions using savepoints

Django’s real implementation is much more complex, but conceptually:
```python
class Atomic:
    def __enter__(self):
        # create a savepoint or start a transaction
        self.savepoint = connection.savepoint()
        return self

    def __exit__(self, exc_type, exc, tb):
        if exc_type is None:
            connection.savepoint_commit(self.savepoint)
        else:
            connection.savepoint_rollback(self.savepoint)
        # if nested atomic blocks exist, they are managed automatically
```
Django wraps database operations inside savepoints so partial work can roll back without affecting outer blocks.

### Question: What Context Manager Executes Your Query?**
---

There are TWO layers:

**Layer 1:** CursorWrapper `context manager(connection.cursor())` When Django talks to the database:

```python
with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM users")
```
cursor() itself is a context manager that ensures:
- cursor is opened
- cursor is closed safely

Django QuerySet uses this under the hood.


**Layer 2**: transaction.atomic() context manager

For transaction handling. Even simple ORM calls like:
```python
User.objects.get(id=1)
```
- CursorWrapper to execute SQL
- If needed, an implicit atomic block (depending on operation)

> So for read operation only cursor context manager is called but for write/delete operation it call both cursor and atomic is called.

**Summary**: Django uses the transaction.atomic() context manager internally for transaction management. It is implemented by the Atomic class in django.db.transaction. All ORM operations use a lower-level context manager, connection.cursor(), which is itself a context manager that ensures safe opening and closing of DB cursors. For write operations, Django automatically wraps them in atomic blocks or savepoints to guarantee consistency.