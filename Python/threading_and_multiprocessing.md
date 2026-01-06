## AsyncIO, Threading and Multiprocessing


### Question: What is thread and process?

In simple terms, a `thread` is like a separate flow of execution within a computer program. Imagine a program as a single kitchen. A thread is like `a single chef` working within that kitchen.

<b>Process</b>

A `process` is like the entire kitchen itself, with all its equipment and resources. It is an independent execution environment. Each process has its own memory space, resources, and a dedicated execution path.

![Process](/Python/images/process.webp)

<b>Thread</b>

A `thread` is like an individual chef within that kitchen. Multiple chefs can work independently within the same kitchen, sharing some resources(memory, interpreter etc) (like ingredients, equipment etc.) but having their own tasks and work areas.

![Thread](/Python/images/thread.webp)


If we wanted to look at a real scenario then we can assume, A program like Microsoft word is a process and inside this word program, running sub-task like searching a text, using find and replace, spelling mistake, grammar mistake is thread. A process you can understand as an independent & active program and a thread is a lightweight process and a part of main process which share memory with other thread and use same resources of process.


<b>Subroutine</b>

We all are familiar with function which is also known as a subroutine, procedure, sub-process, etc. A function is a sequence of instructions packed as a unit to perform a certain task. When the logic of a complex function is divided into several self-contained steps that are themselves functions, then these functions are called helper functions or subroutines.

Subroutines in Python are called by the main function which is responsible for coordinating the use of these subroutines. Subroutines have a single entry point. 

![Subroutine](/Python/images/subroutine.webp)

<b>Coroutines</b>

Coroutines are generalizations of subroutines. They are used for cooperative multitasking where a process voluntarily yield (give away) control periodically or when idle in order to enable multiple applications to be run simultaneously. The difference between coroutine and subroutine is :  

- Unlike subroutines, coroutines have many entry points for suspending and resuming execution. Coroutine can suspend its execution and transfer control to other coroutine and can resume again execution from the point it left off.
- Unlike subroutines, there is no main function to call coroutines in a particular order and coordinate the results. Coroutines are cooperative that means they link together to form a pipeline. One coroutine may consume input data and send it to other that process it. Finally, there may be a coroutine to display the result.

![Coroutines](/Python/images/coroutine.webp)


<b>Coroutine Vs Thread</b>

Now you might be thinking how coroutine is different from threads, both seem to do the same job. 
In the case of threads, it’s an operating system (or run time environment) that switches between threads according to the scheduler. While in the case of a coroutine, it’s the programmer and programming language which decides when to switch coroutines. Coroutines work cooperatively multitask by suspending and resuming at set points by the programmer. 


<b>Python Coroutine</b>

In Python, coroutines are similar to generators but with few extra methods and slight changes in how we use yield statements. Generators produce data for iteration while coroutines can also consume data. 

A coroutine is defined with `async def` and can be pause with `await` or `yield`. They form the basis of `asyncio`.

Threads are managed by the operating system. Coroutines are managed by Python itself (via the event loop). They don’t run in parallel but cooperatively yield control with await so other coroutines can run. This makes them lightweight and efficient.


<b>Event Loop</b>

The event loop is the heart of asyncio.

Think of it like an orchestra conductor. It keeps track of all the tasks (jobs) that need to run, decides which one goes next, and makes sure they cooperate.

When a task is ready, the event loop gives it control. One the task pauses (e.g., waiting for I/O) or finishes, control goes back to the loop, and it moves on to the next task. This cycle continues until there’s no more work, at which point the loop waits quietly until new tasks arrive.

```python
import asyncio

# Create and run an event loop forever
event_loop = asyncio.new_event_loop()
event_loop.run_forever()
```

![Event Loop](/Python/images/event_loop.webp)

Coroutines (Tasks) are our async functions async def. They do some work, and when they hit an await, they reach a yield point. At this point, they hand control back to the event loop.

The event loop is the central scheduler. When a coroutine pauses, the loop notes what it’s waiting for and then goes to check other tasks.

The event loop triggers the actual async work (like waiting for I/O drivers, sleeping, network requests, or database queries). These are typically long-running or blocking operations.

Once the operation is complete (e.g., the data is ready, or the timer finishes), the event loop gets a callback saying, “I’m done”.

The event loop then resumes the coroutine from where it left off, continuing executing until it hits the next await or completes.

```python
import asyncio


async def fetch_data():
    print("Coroutine started: fetching data...")
    # Yield point: coroutine pauses here, event loop takes over
    await asyncio.sleep(2)
    # Event loop will resume this line after 2 seconds
    print("Coroutine resumed: data received!")
    return {"data": 42}


async def main():
    print("Starting main coroutine")
    result = await fetch_data()
    print(f"Result: {result}")

# Run the event loop
asyncio.run(main())

"""
Starting main coroutine
Coroutine started: fetching data...
Coroutine resumed: data received!
Result: {'data': 42}
"""
```
- Coroutines (Tasks) → `fetch_data` and `main` are coroutines.
- Yield Point → `await asyncio.sleep(2)` tells the event loop: "Pause me here."
- Async Operation → `asyncio.sleep(2)` is handled by the event loop (like a timer, or in real life, I/O work such as network or DB).
- Callback → After 2 seconds, the event loop gets the signal: “Sleep is done.”
- Resume → The event loop resumes `fetch_data`, which prints `"Coroutine resumed: data received!"`.


<b>Tasks</b>

A Task is simply a coroutine tied to the event loop. When you turn a coroutine into a Task (usually with `asyncio.create_task()`), it’s automatically scheduled to run in the background by the event loop.

- Tasks also keep a list of callbacks that the event loop will trigger when the task is ready to resume.
- When you create a Task, you’re not adding the Task itself into the loop, instead, a callback to run it is added to the loop’s job list.
- That’s why you should keep a reference to your tasks (e.g., by await task or storing them). If you don’t, Python’s garbage collector might clean them up before they ever get to run.
Creating and awaiting a Task:

```python
import asyncio

async def loudmouth_penguin(magic_number=5):
    await asyncio.sleep(1)
    print(f"Penguin says: {magic_number}!")

async def main():
    # Turn coroutine into a Task and schedule it immediately
    task = asyncio.create_task(loudmouth_penguin(5))
    print("Task created and running in background...")
    
    await task   # Wait for the Task to finish

asyncio.run(main())
print("main() is done!")

"""
Task created and running in background...
Penguin says: 5!
main() is done!
"""
```

Forgetting the Task reference:

```python
import asyncio

async def hello():
    print("hello!")

async def main():
    # BAD: no reference kept to the Task
    asyncio.create_task(hello())
    await asyncio.sleep(0.1)  # event loop may run other tasks first

asyncio.run(main())
```
Here, the "hello!" print might never happen if the Task gets garbage collected before the loop runs it.

<b>Awaitable</b>

Awaitable is the most general category, anything that can be used with await belongs to that.

![Awaitable](/Python/images/awaitable.webp)

Coroutines can be awaited only once.

<b>Future</b>
A Future is an object that represents the status and result of a computation.

Think of it as a status light (red = pending, yellow = running, green = done) for some work that’s happening.

- States → A Future can be `"pending"`, `"cancelled"`, or `"done"`.
- Result → When the state changes to `"done"`, the Future will hold either the computation’s result or an exception.
- Important → A Future itself does not do the computation, it just tracks whether the computation has finished and what came out of it.

In `asyncio`:

- Task is a subclass of Future → when a coroutine finishes, the Task marks itself as "done" and stores the result.

```python
import asyncio


async def main():
    loop = asyncio.get_running_loop()

    # Create a Future object (pending state)
    future = loop.create_future()

    # Simulate completing it later
    loop.call_later(1, future.set_result, "done!")

    print("Waiting for future...")
    result = await future   # Suspends until the Future is marked done
    print("Future result:", result)

asyncio.run(main())

"""
Waiting for future...
Future result: done!
"""
```
Future is a promise about something that will finish later, holding its status and result.

Task is a Future that runs a coroutine until it completes.


### Explain Global Interpreter Lock (GIL) in Python

The Global Interpreter Lock `(GIL)` is a mutual exclusion(`Mutex`) lock used in CPython (the default Python implementation). It ensures that only one thread executes Python bytecode at a time, even if your machine has multiple CPU cores.

CPython uses reference counting for memory management. To update reference counts safely across multiple threads, CPython uses the GIL instead of fine-grained locks. This makes Python:
- simpler
- fast in single-threaded workloads
- safe internally

It limits performance for `CPU-bound multithreading` but does not affect I/O-bound workloads, because I/O operations release the GIL.
For CPU-bound concurrency, we use multiprocessing or C-extensions. For I/O-bound concurrency, threads or asyncio work well.

Django apps are usually I/O-bound:

- DB queries
- HTTP requests
- cache calls
- file operations

But:
- CPU-heavy tasks (e.g., image processing, ML, encryption)
WILL get blocked by the GIL.
- These operations release the GIL → threads run efficiently.


<b>Question Describe the GIL and its effect on multi-threading in Python. How would you solve a CPU-bound task in Python that requires genuine parallelism?</b>

<b>Answer:</b> The Global Interpreter Lock (GIL) is a mutex (mutual exclusion lock) that protects access to Python objects, preventing multiple native threads from executing Python bytecodes simultaneously within a single process.
- Effect: It makes Python's threading module unsuitable for achieving parallelism (running code simultaneously on multiple CPU cores) because only one thread can execute Python code at a time. Threading is still excellent for concurrency in I/O-bound tasks (waiting for network, disk I/O) because the GIL is released during these wait periods.
- Solution for Parallelism: For CPU-bound tasks, you must use the multiprocessing module. This module spawns independent operating system processes, each with its own Python interpreter and memory space, effectively bypassing the GIL and utilizing multiple CPU cores.

```python
import multiprocessing

def cpu_heavy_task():
    # Code that calculates and uses CPU time heavily
    pass

# To achieve parallelism:
if __name__ == '__main__':
    pool = multiprocessing.Pool(processes=4)
    pool.apply_async(cpu_heavy_task)
    pool.close()
    pool.join()
```


### References
1. [Coroutine in Python](https://www.geeksforgeeks.org/python/coroutine-in-python/)
2. [Python Asyncio Tutorial: A Complete Guide
](https://www.lambdatest.com/blog/python-asyncio/)
3. [Python AsyncIO](https://faun.pub/if-you-write-python-you-must-understand-asyncio-b38406114e14)
4. https://www.xanthium.in/creating-threads-sharing-synchronizing-data-using-queue-lock-semaphore-python
