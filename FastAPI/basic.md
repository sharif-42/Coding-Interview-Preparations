## Why FastAPI Is Faster Compared to Django/Flask
FastAPI’s speed advantage comes from architecture, Python async support, and implementation choices.

### ASGI vs WSGI (Async vs Sync)
**WSGI (Django, Flask)**
- Designed for synchronous applications.
- Each request is handled one at a time per worker.
- To handle multiple requests, you must run many workers (processes or threads).
- Blocking I/O (DB calls, HTTP calls, file operations) blocks the entire worker.

**ASGI (FastAPI)**
- Designed for asynchronous Python web apps.
- Integrated support for async/await, enabling:
    - Non-blocking I/O
    - Thousands of simultaneous connections using a single worker
- Built for modern needs: WebSockets, streaming, background tasks.

> **ASGI allows FastAPI to scale with fewer resources and handle more concurrent requests.**

### Uvicorn vs Gunicorn
**Gunicorn (Django/Flask default)**
- Mature WSGI server.
- Blocking and synchronous.
- Uses multiple workers to achieve concurrency → heavy CPU + RAM usage.

**Uvicorn (FastAPI default ASGI server)**
- Lightning-fast event loop powered by uvloop (written in C).
- Async I/O → handles thousands of connections efficiently.
- Reduced context switching → less overhead than WSGI.

>**FastAPI + Uvicorn processes requests using a high-performance event loop instead of threads.**

### Concurrency With async/await

**in FastAPI**
```python
async def get_user():
    await db.fetch(...)
```
While waiting for DB/network:
- The event loop continues processing other requests.

**In Django/Flask:**
```python
def get_user():
    user = db.fetch(...)  # blocks the worker
```
While waiting for DB:
- The worker is blocked.
- No other requests can be processed by that worker.

> **FastAPI can handle 10,000+ concurrent connections with ease (like Node.js or Go).**

### Lightweight Framework Design

**Django:**
- Heavy ORM
- Heavy middleware stack
- Tight coupling between components
- Not optimized for microservices or performance-first systems

**Flask:**
- Lightweight, but synchronous and not optimized for speed.

**FastAPI:**
- Minimal overhead
- Modular
- Async-native design

>**FastAPI routes execute with minimal framework overhead.**

### Starlette + Pydantic Under the Hood
FastAPI is built on:

**Starlette**
- A minimal, high-performance ASGI framework.

**Pydantic**
- Extremely fast validation (optimized with Rust implementation in v2).

These provide:
- Fast request parsing
- Automatic validation
- Less overhead than Django’s middleware-heavy request pipeline

> **Request parsing, validation, routing are all faster.**