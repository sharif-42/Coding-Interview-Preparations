## AsyncIO, Threading and Multiprocessing

### Concurrency vs Parallelism
---

```
┌──────────────────────────────────────────────────────────────┐
│                  CONCURRENCY vs PARALLELISM                    │
│                                                              │
│   Concurrency (one cook, multiple dishes)                     │
│   ───────────────────────────────────────                  │
│   Task A: ███░░░██░░░░███   (switch between tasks)          │
│   Task B: ░░░███░░████░░░   Tasks interleave on 1 core     │
│                                                              │
│   Parallelism (multiple cooks, multiple dishes)               │
│   ───────────────────────────────────────                  │
│   Core 1 → Task A: ██████████   (truly simultaneous)       │
│   Core 2 → Task B: ██████████   Tasks run on separate cores  │
└──────────────────────────────────────────────────────────────┘
```

- **Concurrency**: Dealing with multiple things at once (structure). One worker switching between tasks.
- **Parallelism**: Doing multiple things at once (execution). Multiple workers running simultaneously.

### Python's Three Concurrency Models
---

```
┌──────────────────────────────────────────────────────────────┐
│           CHOOSING THE RIGHT CONCURRENCY MODEL                │
│                                                              │
│           ┌─────────────────┐                                 │
│           │  What type of   │                                 │
│           │  work is it?    │                                 │
│           └────────┬────────┘                                 │
│                   │                                          │
│       ┌───────────┼───────────┐                            │
│       ▼           ▼           ▼                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────────────┐              │
│  │ I/O-    │ │ I/O-    │ │ CPU-bound         │              │
│  │ bound   │ │ bound   │ │ (math, data)      │              │
│  │ (many)  │ │ (few)   │ │                   │              │
│  └────┬────┘ └────┬────┘ └────────┬────────┘              │
│       ▼          ▼              ▼                            │
│   asyncio    threading    multiprocessing                   │
│  (1 thread)  (GIL-bound)  (separate processes)              │
│   ○ ○ ○ ○     ○ ○ ○         ● ● ● ●                        │
│  cooperative  OS-managed   true parallelism                  │
│                                                              │
│  ○ = coroutine   ○ = thread   ● = process (own memory)       │
└──────────────────────────────────────────────────────────────┘
```

| Feature | `asyncio` | `threading` | `multiprocessing` |
| --- | --- | --- | --- |
| **Best for** | I/O-bound (many connections) | I/O-bound (simpler code) | CPU-bound (computation) |
| **Parallelism** | No (single thread) | No (GIL) | Yes (separate processes) |
| **Concurrency** | Yes (cooperative) | Yes (preemptive) | Yes (true parallel) |
| **Memory** | Shared (lightweight) | Shared (same process) | Separate (each process) |
| **Overhead** | Very low | Low-medium | High (process creation) |
| **Switching** | Developer controls (`await`) | OS controls | OS controls |
| **Max tasks** | 10,000s+ easily | 100s-1000s | Limited by CPU cores |
| **Race conditions** | Rare (single thread) | Possible (need locks) | Less common (separate memory) |
| **Python GIL** | N/A (single thread) | Limits CPU work | Bypasses GIL |

### Question: What is thread and process?

In simple terms, a `thread` is like a separate flow of execution within a computer program. Imagine a program as a single kitchen. A thread is like `a single chef` working within that kitchen.

<b>Process</b>

A `process` is like the entire kitchen itself, with all its equipment and resources. It is an independent execution environment. Each process has its own memory space, resources, and a dedicated execution path.

![Process](/Python/images/process.webp)

<b>Thread</b>

A `thread` is like an individual chef within that kitchen. Multiple chefs can work independently within the same kitchen, sharing some resources(memory, interpreter etc) (like ingredients, equipment etc.) but having their own tasks and work areas.

![Thread](/Python/images/thread.webp)


If we wanted to look at a real scenario then we can assume, A program like Microsoft Word is a process and inside this Word program, running sub-tasks like searching a text, using find and replace, spelling check, grammar check are threads. A process you can understand as an independent & active program and a thread is a lightweight process and a part of the main process which shares memory with other threads and uses the same resources of the process.

**Process vs Thread Comparison:**

```
┌──────────────────────────────────────────────────────────────┐
│                    PROCESS vs THREAD                          │
│                                                              │
│   Process A                    Process B                     │
│   ┌──────────────────┐       ┌──────────────────┐         │
│   │  Own Memory       │       │  Own Memory       │         │
│   │  Own GIL          │       │  Own GIL          │         │
│   │                  │       │                  │         │
│   │  ┌──────┐┌─────┐│       │  ┌──────┐┌─────┐│         │
│   │  │ Th 1 ││ Th 2││       │  │ Th 1 ││ Th 2││         │
│   │  └──────┘└─────┘│       │  └──────┘└─────┘│         │
│   │  (share memory)  │       │  (share memory)  │         │
│   └──────────────────┘       └──────────────────┘         │
│     (isolated from B)           (isolated from A)          │
│                                                              │
│   Threads within a process:  share memory, heap, code        │
│   Processes are isolated:    separate memory, own GIL         │
└──────────────────────────────────────────────────────────────┘
```

| Feature | Process | Thread |
| --- | --- | --- |
| Memory | Own isolated memory | Shared memory within process |
| Creation overhead | Heavy (new interpreter) | Lightweight |
| Communication | IPC (pipes, queues, sockets) | Direct (shared variables) |
| Crash impact | One process crash doesn't affect others | One thread crash can kill the process |
| GIL | Each process has its own GIL | All threads share one GIL |
| Best for | CPU-bound parallelism | I/O-bound concurrency |


<b>Subroutine</b>

We all are familiar with function which is also known as a subroutine, procedure, sub-process, etc. A function is a sequence of instructions packed as a unit to perform a certain task. When the logic of a complex function is divided into several self-contained steps that are themselves functions, then these functions are called helper functions or subroutines.

Subroutines in Python are called by the main function which is responsible for coordinating the use of these subroutines. Subroutines have a single entry point. 

![Subroutine](/Python/images/subroutine.webp)

<b>Coroutines</b>

Coroutines are generalizations of subroutines. They are used for cooperative multitasking where a process voluntarily yield (give away) control periodically or when idle in order to enable multiple applications to be run simultaneously. The difference between coroutine and subroutine is :  

- Unlike subroutines, coroutines have many entry points for suspending and resuming execution. Coroutine can suspend its execution and transfer control to other coroutine and can resume again execution from the point it left off.
- Unlike subroutines, there is no main function to call coroutines in a particular order and coordinate the results. Coroutines are cooperative that means they link together to form a pipeline. One coroutine may consume input data and send it to other that process it. Finally, there may be a coroutine to display the result.

![Coroutines](/Python/images/coroutine.webp)

**Subroutine vs Coroutine Comparison:**

```
  Subroutine:                     Coroutine:
  ┌───────────┐                   ┌───────────┐
  │  start    │                   │  start    │
  │  │        │                   │  │        │
  │  │ run    │                   │  │ run    │
  │  │        │                   │  │        │
  │  ▼        │                   │  ▼ await  │ ──► pauses, gives control
  │  end      │                   │  │        │
  └───────────┘                   │  │ resume │ ◄── gets control back
  (one entry, one exit)            │  │        │
                                   │  ▼ await  │ ──► pauses again
                                   │  │        │
                                   │  ▼        │
                                   │  end      │
                                   └───────────┘
                                   (multiple entry/exit points)
```


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

When a task is ready, the event loop gives it control. Once the task pauses (e.g., waiting for I/O) or finishes, control goes back to the loop, and it moves on to the next task. This cycle continues until there's no more work, at which point the loop waits quietly until new tasks arrive.

```python
import asyncio

# Create and run an event loop forever
event_loop = asyncio.new_event_loop()
event_loop.run_forever()
```

![Event Loop](/Python/images/event_loop.webp)

**Event Loop Cycle Diagram:**
```
┌─────────────────────────────────────────────────────────┐
│                 EVENT LOOP CYCLE                           │
│                                                           │
│   ┌───────────────────────────────┐                       │
│   │                               │                       │
│   ▼                               │                       │
│   Check ready tasks               │                       │
│   │                               │                       │
│   ▼                               │                       │
│   Run coroutine until await       │                       │
│   │                               │                       │
│   ▼                               │                       │
│   Coroutine hits await             │                       │
│   (yields control back)            │                       │
│   │                               │                       │
│   ▼                               │                       │
│   Register I/O callback            │                       │
│   │                               │                       │
│   ▼                               │                       │
│   Check for completed I/O          │                       │
│   │                               │                       │
│   ▼                               │                       │
│   Resume completed coroutines ────┘                       │
│                                                           │
│   Repeat until no more tasks remain                       │
└─────────────────────────────────────────────────────────┘
```

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

**Interview tip:** The GIL is a CPython implementation detail, not a Python language feature. Other implementations like Jython (Java) and IronPython (.NET) don't have a GIL. Python 3.13+ introduces an experimental "free-threaded" mode (`--disable-gil`) to remove this limitation.

**GIL Impact Diagram:**
```
┌──────────────────────────────────────────────────────────────┐
│                  GIL IMPACT ON THREADS                        │
│                                                              │
│  CPU-bound threading (GIL hurts):                             │
│  Thread 1: ███░░░███░░░███   only 1 runs at a time            │
│  Thread 2: ░░░███░░░███░░░   → NO speedup!                   │
│                                                              │
│  I/O-bound threading (GIL is fine):                           │
│  Thread 1: █░░░░░░░░░█░░░░   █ = Python code running         │
│  Thread 2: ░█░░░░░░░░░█░░░   ░ = waiting for I/O             │
│  Thread 3: ░░█░░░░░░░░░█░░   GIL released during I/O wait   │
│  → Other threads can run while one waits!                    │
│                                                              │
│  multiprocessing (bypasses GIL):                              │
│  Process 1: ███████████████   own GIL, own memory            │
│  Process 2: ███████████████   true parallelism              │
│  Process 3: ███████████████   on separate CPU cores          │
└──────────────────────────────────────────────────────────────┘
```

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

### Threading vs Multiprocessing vs AsyncIO — Code Comparison
---

**Same task: Fetch 5 URLs**

**1. Threading (I/O-bound, simple):**
```python
import threading
import requests

def fetch(url):
    response = requests.get(url)
    print(f"{url}: {response.status_code}")

urls = ["https://example.com"] * 5

threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

**2. Multiprocessing (CPU-bound):**
```python
from multiprocessing import Pool

def compute(n):
    return sum(i * i for i in range(n))

if __name__ == '__main__':
    with Pool(4) as pool:
        results = pool.map(compute, [10**7] * 4)
    print(results)
```

**3. AsyncIO (I/O-bound, high scale):**
```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, "https://example.com") for _ in range(5)]
        results = await asyncio.gather(*tasks)
        print(f"Fetched {len(results)} pages")

asyncio.run(main())
```

**Performance comparison for 100 HTTP requests:**
```
┌─────────────┬─────────────┬─────────────┬─────────────┐
│  Approach   │  Time (100   │  Memory     │  Complexity  │
│             │  requests)   │             │             │
├─────────────┼─────────────┼─────────────┼─────────────┤
│ Sequential  │  ~100s       │  Low        │  Simple     │
│ Threading   │  ~2s         │  Medium     │  Medium     │
│ AsyncIO     │  ~1.5s       │  Low        │  Medium-High│
│ Multiproc   │  ~3s         │  High       │  Medium     │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

### Thread Synchronization — Avoiding Race Conditions
---

When threads share data, race conditions can occur:

```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(1_000_000):
        counter += 1   # NOT thread-safe!

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()

print(counter)  # Expected: 2,000,000 — Actual: ~1,500,000 (race condition!)
```

**Fix with Lock:**
```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(1_000_000):
        with lock:           # only one thread at a time
            counter += 1

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()

print(counter)  # 2,000,000 ✓
```

**Synchronization primitives:**

| Primitive | Purpose |
| --- | --- |
| `Lock` | Mutual exclusion — one thread at a time |
| `RLock` | Reentrant lock — same thread can acquire multiple times |
| `Semaphore` | Allow up to N threads concurrently |
| `Event` | Signal between threads (one sets, others wait) |
| `Condition` | Wait for a specific condition to be true |
| `Queue` | Thread-safe data sharing (producer-consumer) |

### `asyncio.gather()` vs `asyncio.create_task()` vs `asyncio.wait()`
---

```python
import asyncio

async def task(name, delay):
    await asyncio.sleep(delay)
    return f"{name} done"

# gather — run multiple coroutines, get all results in order
async def main_gather():
    results = await asyncio.gather(
        task("A", 2),
        task("B", 1),
        task("C", 3),
    )
    print(results)  # ['A done', 'B done', 'C done'] (in order)

# create_task — schedule individually, more control
async def main_tasks():
    t1 = asyncio.create_task(task("A", 2))
    t2 = asyncio.create_task(task("B", 1))
    # Both running concurrently now
    result1 = await t1
    result2 = await t2

# wait — more control over completion
async def main_wait():
    tasks = [asyncio.create_task(task(n, i)) for i, n in enumerate("ABC")]
    done, pending = await asyncio.wait(tasks, timeout=2)
    # done = completed tasks, pending = still running
```

| Method | Returns | Order | Error handling | Use case |
| --- | --- | --- | --- | --- |
| `gather()` | List of results | Input order | One failure can cancel all | Run N tasks, get all results |
| `create_task()` | Individual Task | N/A | Per-task | Fine-grained control |
| `wait()` | (done, pending) sets | Completion order | Per-task | Timeouts, partial results |

---

### Common Interview Questions
---

**Q: What is the difference between threading, multiprocessing, and asyncio in Python?**
- **Threading**: Multiple threads in one process, share memory, limited by GIL for CPU work. Good for I/O-bound tasks.
- **Multiprocessing**: Separate processes with own memory and GIL. True parallelism. Good for CPU-bound tasks.
- **AsyncIO**: Single-threaded event loop with coroutines. Most lightweight. Best for high-concurrency I/O (thousands of connections).

**Q: When would you use asyncio over threading?**
When you have many I/O-bound tasks (hundreds or thousands of concurrent connections). AsyncIO uses coroutines which are much lighter than threads (~120 bytes vs ~8 KB per thread). Threading is simpler for fewer concurrent tasks.

**Q: What is the difference between `async def` and `def`?**
`async def` defines a coroutine function. Calling it returns a coroutine object (not a result). You must `await` it or pass it to the event loop. Regular `def` runs synchronously and returns a result immediately.

**Q: What is `await` and when can you use it?**
`await` can only be used inside an `async def` function. It pauses the coroutine and gives control back to the event loop, allowing other coroutines to run while this one waits for an I/O operation.

**Q: How does `asyncio.gather()` differ from sequential awaiting?**
```python
# Sequential — total time: 3 seconds
result1 = await task_1s()   # waits 1s
result2 = await task_2s()   # waits 2s

# Concurrent — total time: 2 seconds (max of individual times)
result1, result2 = await asyncio.gather(task_1s(), task_2s())
```

**Q: What is a race condition and how do you prevent it?**
A race condition occurs when multiple threads access shared data simultaneously and the result depends on thread scheduling. Prevent with `Lock`, `Queue`, or by avoiding shared mutable state. With asyncio, race conditions are rare because only one coroutine runs at a time (single-threaded).

**Q: How does Django/Flask handle concurrency?**
- **Django**: Uses WSGI (synchronous), one thread per request. Supports async views since Django 3.1+ with ASGI.
- **Flask**: Synchronous by default, uses threading for concurrent requests.
- **FastAPI**: Built on asyncio + ASGI, natively supports `async def` handlers for high concurrency.

---


### References
1. [Coroutine in Python](https://www.geeksforgeeks.org/python/coroutine-in-python/)
2. [Python Asyncio Tutorial: A Complete Guide
](https://www.lambdatest.com/blog/python-asyncio/)
3. [Python AsyncIO](https://faun.pub/if-you-write-python-you-must-understand-asyncio-b38406114e14)
4. [Creating and Sharing data between Python threads for the Absolute Beginner](https://www.xanthium.in/creating-threads-sharing-synchronizing-data-using-queue-lock-semaphore-python)
