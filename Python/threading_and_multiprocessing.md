### Threading and Multiprocessing
---

<b>Question: What is thread and process?</b>


<b>Answer</b>: In simple terms, a `thread` is like a separate flow of execution within a computer program. Imagine a program as a single kitchen. A thread is like `a single chef` working within that kitchen.

A `process` is like the entire kitchen itself, with all its equipment and resources. It is an independent execution environment. Each process has its own memory space, resources, and a dedicated execution path.

A `thread` is like an individual chef within that kitchen. Multiple chefs can work independently within the same kitchen, sharing some resources (like ingredients, equipment etc.) but having their own tasks and work areas.

If we wanted to look at a real scenario then we can assume, A program like Microsoft word is a process and inside this word program, running sub-task like searching a text, using find and replace, spelling mistake, grammar mistake is thread. A process you can understand as an independent & active program and a thread is a lightweight process and a part of main process which share memory with other thread and use same resources of process.

<b>Question: Explain Global Interpreter Lock (GIL) in Python </b>


<b>Answer</b>: The Global Interpreter Lock (GIL) is a mutual exclusion(`Mutex`) lock used in CPython (the default Python implementation). It ensures that only one thread executes Python bytecode at a time, even if your machine has multiple CPU cores.

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
