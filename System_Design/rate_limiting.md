# Rate Limiting — From Django to FastAPI to API Gateway

Imagine you own a popular restaurant.

One evening, a single customer walks in and starts ordering everything on the menu — appetizers, mains, desserts, drinks — all at once. The kitchen gets overwhelmed. Other customers wait forever. Some leave.

You need a rule: **each customer can place a maximum of 3 orders per 10 minutes**.

That rule is **rate limiting**.

In software, rate limiting controls how many requests a client (user, IP address, API key) can make to your server within a given time window. It protects your application from abuse, brute-force attacks, accidental traffic spikes, and ensures fair usage across all clients.

Let's start with how rate limiting works in Django, then see how FastAPI handles it differently, and finally scale the concept up to a real-world API Gateway managing 5 microservices.

---

## 1. Rate Limiting in Django

Django does not ship with built-in rate limiting out of the box. But the ecosystem gives you two excellent options:

| Approach | Library | Best For |
|---|---|---|
| Django REST Framework Throttling | `djangorestframework` | API-based Django projects using DRF |
| Django Ratelimit (decorator-based) | `django-ratelimit` | Any Django view — class-based or function-based |

We will cover both in depth.

---

### 1.1 DRF Throttling — The Built-In Way

If you are using Django REST Framework (which most production Django APIs do), throttling is built right in. No extra packages needed.

#### How It Works

DRF throttling works like a **bouncer at a nightclub**.

The bouncer has a clipboard. Every time you enter, he marks the time. If you try to enter more than the allowed number of times within the time window, he says: *"Sorry, come back later."*

Internally, DRF:
1. Identifies the client (by IP address for anonymous users, by user ID for authenticated users).
2. Checks a cache backend (Redis, Memcached, or local memory) for how many requests this client has made recently.
3. If the count exceeds the limit → returns `429 Too Many Requests`.
4. If not → allows the request and increments the counter.

#### Step-by-Step Configuration

**Step 1: settings.py — Define Global Throttle Rates**

```python
# settings.py

REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '20/minute',     # Anonymous users: 20 requests per minute
        'user': '100/minute',    # Authenticated users: 100 requests per minute
    }
}
```

This applies globally to **every API view** in your project.

**Step 2: How DRF Identifies Clients**

| Throttle Class | Identifies By | Use Case |
|---|---|---|
| `AnonRateThrottle` | Client IP address (`REMOTE_ADDR`) | Public endpoints, login pages |
| `UserRateThrottle` | Authenticated user ID | Logged-in user actions |
| `ScopedRateThrottle` | A custom scope string you define | Different limits for different endpoints |

**Step 3: Per-View Throttle Override**

You do not always want the same limit everywhere. A login endpoint should be stricter than a product listing.

```python
# views.py
from rest_framework.throttling import AnonRateThrottle, UserRateThrottle
from rest_framework.views import APIView
from rest_framework.response import Response


class LoginView(APIView):
    """
    Strict rate limit on login to prevent brute-force attacks.
    """
    throttle_classes = [AnonRateThrottle]
    throttle_scope = 'login'

    def post(self, request):
        # ... authentication logic
        return Response({"message": "Login successful"})


class ProductListView(APIView):
    """
    Relaxed limit — users browse products frequently.
    """
    throttle_classes = [UserRateThrottle]

    def get(self, request):
        # ... return product list
        return Response({"products": []})
```

Then in settings:

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'anon': '20/minute',
        'user': '100/minute',
        'login': '5/minute',   # Only 5 login attempts per minute
    }
}
```

**Step 4: Custom Throttle Class**

Sometimes you need logic that DRF does not provide out of the box. For example, rate limiting by API key instead of user ID.

```python
# throttles.py
from rest_framework.throttling import SimpleRateThrottle


class APIKeyThrottle(SimpleRateThrottle):
    """
    Rate limit based on API key in the request header.
    Different API keys can have different tiers.
    """
    scope = 'api_key'

    def get_cache_key(self, request, view):
        api_key = request.headers.get('X-API-Key')
        if not api_key:
            return self.get_ident(request)  # Fall back to IP
        return self.cache_format % {
            'scope': self.scope,
            'ident': api_key,
        }

    def get_rate(self):
        """
        Override to support tiered rate limits.
        Premium keys get higher limits.
        """
        # You could look up the key's tier from a database here
        return '500/hour'
```

#### What Happens When a Client Is Throttled?

DRF returns a `429 Too Many Requests` response:

```json
{
    "detail": "Request was throttled. Expected available in 42 seconds."
}
```

The response also includes a `Retry-After` header:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 42
Content-Type: application/json
```

#### DRF Throttle — Under the Hood

Here is what happens on every single request:

```
Client Request
      │
      ▼
┌─────────────────────────┐
│   DRF Middleware Chain   │
│   (Authentication first)│
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Throttle Check        │
│                         │
│  1. Identify client     │
│     (IP / User ID /     │
│      API Key / Scope)   │
│                         │
│  2. Check cache:        │
│     "How many requests  │
│      in the last window?"│
│                         │
│  3. Compare to limit    │
└────────────┬────────────┘
             │
        ┌────┴────┐
        │         │
     Under      Over
     Limit      Limit
        │         │
        ▼         ▼
   Allow req   Return 429
   + increment  + Retry-After
     counter     header
```

#### Cache Backend Matters

DRF throttling relies on Django's cache framework. In production, **always use Redis or Memcached**:

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}
```

> **Why not local memory?** In production, you typically run multiple Django processes (via Gunicorn or uWSGI). Local memory cache is **per-process** — each process has its own counter. A user could effectively multiply their rate limit by the number of processes. Redis is shared across all processes, so the count is accurate.

---

### 1.2 Django Ratelimit — The Decorator Approach

If you are not using DRF — or you want rate limiting on regular Django views — the `django-ratelimit` package is excellent.

```bash
pip install django-ratelimit
```

#### Basic Usage

```python
# views.py
from django.http import JsonResponse
from django_ratelimit.decorators import ratelimit


@ratelimit(key='ip', rate='10/m', method='POST', block=True)
def contact_form(request):
    """
    Allow 10 form submissions per minute per IP.
    block=True means return 403 automatically when exceeded.
    """
    # ... process form
    return JsonResponse({"message": "Message sent!"})


@ratelimit(key='user', rate='5/m', method='POST', block=True)
def change_password(request):
    """
    Authenticated users: 5 password change attempts per minute.
    """
    # ... change password logic
    return JsonResponse({"message": "Password changed"})
```

#### Key Options

| Parameter | Options | Description |
|---|---|---|
| `key` | `'ip'`, `'user'`, `'user_or_ip'`, `'header:X-API-Key'` | What to identify the client by |
| `rate` | `'10/m'`, `'100/h'`, `'1000/d'` | Limit per time window (m=minute, h=hour, d=day) |
| `method` | `'GET'`, `'POST'`, `'ALL'`, `ratelimit.ALL` | Which HTTP methods to limit |
| `block` | `True` / `False` | If True → auto-return 403. If False → set `request.limited = True` and let the view decide |

#### Handling Gracefully (block=False)

```python
@ratelimit(key='ip', rate='10/m', method='POST', block=False)
def search(request):
    if request.limited:
        return JsonResponse(
            {"error": "Too many searches. Please slow down."},
            status=429
        )
    # ... perform search
    return JsonResponse({"results": []})
```

#### Group-Based Limits

You can group multiple views under a single shared counter:

```python
@ratelimit(key='user', rate='20/m', group='api_writes')
def create_post(request):
    ...

@ratelimit(key='user', rate='20/m', group='api_writes')
def create_comment(request):
    ...
```

Now `create_post` and `create_comment` **share the same 20/min budget**. If a user creates 15 posts, they only have 5 comments left in that window.

---

### 1.3 Django Rate Limiting — Comparison Table

| Feature | DRF Throttling | django-ratelimit |
|---|---|---|
| **Requires DRF?** | Yes | No |
| **Configuration** | settings.py + class attributes | Decorators on views |
| **Identifies by** | IP, User ID, custom | IP, User, Header, custom |
| **Cache backend** | Django cache (Redis recommended) | Django cache (Redis recommended) |
| **Response when exceeded** | 429 + `Retry-After` header | 403 (or custom with `block=False`) |
| **Scoped limits** | Yes (`ScopedRateThrottle`) | Yes (`group` parameter) |
| **Per-view override** | Yes | Yes (each decorator is per-view) |
| **Best for** | DRF-based APIs | Any Django project |

---

### 1.4 Common Rate Limiting Algorithms

Before we move to the API Gateway section, it is important to understand the algorithms behind rate limiting. These same algorithms power both Django's throttling and API gateway rate limiting.

#### 1. Fixed Window Counter

The simplest approach. Divide time into fixed windows (e.g., 1-minute intervals) and count requests per window.

```
Window: 10:00 - 10:01    Limit: 5 requests
──────────────────────────────────────────
│ ✓ │ ✓ │ ✓ │ ✓ │ ✓ │ ✗ │ ✗ │ ✗ │
──────────────────────────────────────────
  1   2   3   4   5   BLOCKED

Window: 10:01 - 10:02    Counter resets!
──────────────────────────────────────────
│ ✓ │ ✓ │ ...
──────────────────────────────────────────
  1   2
```

**Problem:** A user can send 5 requests at 10:00:59 and 5 more at 10:01:01 — that is 10 requests in 2 seconds, even though the limit is 5/minute.

#### 2. Sliding Window Log

Instead of fixed windows, keep a log of every request timestamp. For each new request, count how many requests occurred in the last N seconds.

```
Limit: 5 requests per 60 seconds

Request at 10:00:30 → Log: [10:00:30]                    → Count: 1 ✓
Request at 10:00:45 → Log: [10:00:30, 10:00:45]          → Count: 2 ✓
Request at 10:01:10 → Log: [10:00:30, 10:00:45, 10:01:10]→ Count: 3 ✓
                       ↑ Check: remove entries older than (10:01:10 - 60s = 10:00:10)
                       10:00:30 stays (it is after 10:00:10)
```

**Accurate but expensive** — stores every timestamp. Memory grows with traffic.

#### 3. Sliding Window Counter

A hybrid. Combines the efficiency of fixed windows with the accuracy of sliding windows.

```
Current time: 10:01:15 (15 seconds into the current window)

Previous window (10:00 - 10:01): 12 requests
Current window  (10:01 - 10:02): 3 requests so far

Weighted count = (12 × 0.75) + (3 × 1.0)
               = 9 + 3
               = 12
               ↑
               75% of the previous window overlaps with
               our 60-second sliding window
```

**This is what DRF uses internally.**

#### 4. Token Bucket

Imagine a bucket that holds tokens. Every request costs 1 token. Tokens refill at a constant rate.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

Time 0s:  [🪙🪙🪙🪙🪙🪙🪙🪙🪙🪙]  10 tokens — full
           Client sends 5 requests
Time 1s:  [🪙🪙🪙🪙🪙⚪⚪⚪⚪⚪]   5 tokens left
           +2 refilled
Time 2s:  [🪙🪙🪙🪙🪙🪙🪙⚪⚪⚪]   7 tokens
           Client sends burst of 7
Time 3s:  [⚪⚪⚪⚪⚪⚪⚪⚪⚪⚪]   0 tokens — BLOCKED
           +2 refilled
Time 4s:  [🪙🪙⚪⚪⚪⚪⚪⚪⚪⚪]   2 tokens — can send 2
```

**Allows bursts** while maintaining an average rate. Used by AWS API Gateway, Stripe, and many others.

#### 5. Leaky Bucket

Like the token bucket, but reversed. Requests fill a bucket, and the bucket "leaks" (processes) at a constant rate.

```
Bucket capacity: 5
Process rate: 1 request/second

Incoming: ████████ (8 requests arrive at once)

Bucket:   [█][█][█][█][█] ← Full! 3 requests DROPPED
           │
           ▼ (leaks at 1/sec)
        Processed: █ ... █ ... █ ... █ ... █
```

**Smooths out bursts.** Used by Nginx (`limit_req` with `burst`).

#### Algorithm Comparison

| Algorithm | Accuracy | Memory | Burst Handling | Used By |
|---|---|---|---|---|
| Fixed Window | Low | Very low | Allows boundary bursts | Simple APIs |
| Sliding Window Log | High | High | No bursts | Strict rate limiters |
| Sliding Window Counter | High | Low | Minimal bursts | DRF, Kong, Envoy |
| Token Bucket | High | Low | Allows controlled bursts | AWS API Gateway, Stripe |
| Leaky Bucket | High | Low | Smooths bursts | Nginx, HAProxy |

---

## 2. Rate Limiting in FastAPI

FastAPI does not ship with built-in rate limiting either. But unlike Django, FastAPI's async-first architecture and middleware system give you different — and in many ways more flexible — options.

| Approach | Library | Best For |
|---|---|---|
| SlowAPI (most popular) | `slowapi` | Drop-in rate limiting with decorator syntax |
| Custom Middleware + Redis | `redis` / `aioredis` | Full control, production-grade, async-native |
| FastAPI Dependencies | Built-in `Depends()` | Lightweight, no extra packages |

---

### 2.1 SlowAPI — The Most Popular Option

SlowAPI is the FastAPI equivalent of `django-ratelimit`. It is built on top of the battle-tested `limits` library and integrates cleanly with FastAPI.

```bash
pip install slowapi
```

#### How It Works

SlowAPI works like a **toll booth on a highway**.

Every car (request) passes through. The toll booth records the license plate (IP / user ID / API key) and checks: *"Has this car passed through too many times in the last 10 minutes?"* If yes — the barrier stays down.

Internally, SlowAPI:
1. Intercepts the request before it reaches your endpoint.
2. Extracts a key (IP address, user ID, custom function).
3. Checks an in-memory store or Redis for the current count.
4. If over the limit → returns `429 Too Many Requests`.
5. If under → allows the request and increments the counter.

#### Step-by-Step Setup

**Step 1: Create the Limiter**

```python
# main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# Create the limiter — uses client IP as the default key
limiter = Limiter(key_func=get_remote_address)

app = FastAPI(title="My API")

# Attach limiter to the app state
app.state.limiter = limiter

# Register the 429 error handler
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

**Step 2: Apply Limits to Endpoints**

```python
@app.get("/api/products")
@limiter.limit("60/minute")
async def list_products(request: Request):
    """
    Relaxed limit — users browse products frequently.
    60 requests per minute per IP.
    """
    return {"products": ["Widget A", "Widget B"]}


@app.post("/api/login")
@limiter.limit("5/minute")
async def login(request: Request):
    """
    Strict limit on login to prevent brute-force.
    Only 5 attempts per minute per IP.
    """
    return {"message": "Login successful"}


@app.post("/api/orders")
@limiter.limit("20/minute")
async def create_order(request: Request):
    """
    Moderate limit — orders are expensive operations.
    """
    return {"order_id": "ORD-12345"}
```

> **Important:** The `request: Request` parameter is **required** in every rate-limited endpoint. SlowAPI needs access to the request object to extract the client identifier. If you forget it, you will get a confusing error.

**Step 3: Multiple Limits on One Endpoint**

You can stack multiple rate limits:

```python
@app.get("/api/search")
@limiter.limit("30/minute")   # Short-term burst protection
@limiter.limit("500/hour")    # Long-term abuse protection
@limiter.limit("5000/day")    # Daily quota
async def search(request: Request, q: str):
    return {"results": []}
```

All three limits are checked. The **first one to be exceeded** triggers the 429.

**Step 4: Redis Backend for Production**

By default, SlowAPI uses in-memory storage. In production with multiple workers (Uvicorn + Gunicorn), you **must** use Redis:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://127.0.0.1:6379/0",  # Redis backend
)
```

> Same reason as Django — in-memory storage is per-process. With 4 Uvicorn workers, a user effectively gets 4x the rate limit. Redis is shared across all workers.

#### Custom Key Functions

The real power of SlowAPI is custom key functions. You can rate limit by anything:

```python
from fastapi import Request, Header


def get_api_key(request: Request) -> str:
    """Rate limit by API key from header."""
    api_key = request.headers.get("X-API-Key", "anonymous")
    return api_key


def get_user_id(request: Request) -> str:
    """Rate limit by authenticated user ID."""
    # Assuming you have auth middleware that sets request.state.user_id
    return getattr(request.state, "user_id", get_remote_address(request))


# Use different key functions per endpoint
@app.get("/api/public/data")
@limiter.limit("100/minute", key_func=get_api_key)
async def public_data(request: Request):
    return {"data": "public"}


@app.post("/api/user/settings")
@limiter.limit("10/minute", key_func=get_user_id)
async def update_settings(request: Request):
    return {"message": "Settings updated"}
```

#### Dynamic Limits Based on User Tier

You can return different limits based on the user:

```python
def dynamic_limit(request: Request) -> str:
    """
    Return different rate limits based on user tier.
    This function returns the LIMIT STRING, not the key.
    """
    api_key = request.headers.get("X-API-Key", "")

    # Look up the tier (in production, cache this in Redis)
    tier = get_tier_from_api_key(api_key)  # returns 'free', 'pro', or 'enterprise'

    limits = {
        "free": "20/minute",
        "pro": "200/minute",
        "enterprise": "2000/minute",
    }
    return limits.get(tier, "10/minute")  # Default: 10/min


@app.get("/api/data")
@limiter.limit(dynamic_limit)
async def get_data(request: Request):
    return {"data": "here"}
```

---

### 2.2 Custom Middleware — Full Control

If you want complete control without a third-party library, you can build rate limiting as FastAPI middleware using Redis directly.

```python
# middleware/rate_limiter.py
import time
import redis.asyncio as redis
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse


class RateLimitMiddleware(BaseHTTPMiddleware):
    """
    Sliding window rate limiter using Redis.
    Applies globally to all endpoints.
    """

    def __init__(self, app, redis_url: str = "redis://localhost:6379/0",
                 limit: int = 100, window: int = 60):
        super().__init__(app)
        self.redis = redis.from_url(redis_url)
        self.limit = limit      # Max requests
        self.window = window    # Time window in seconds

    async def dispatch(self, request: Request, call_next) -> Response:
        # Identify the client
        client_ip = request.client.host
        key = f"ratelimit:{client_ip}:{request.url.path}"

        # Sliding window using Redis sorted set
        now = time.time()
        window_start = now - self.window

        pipe = self.redis.pipeline()
        # Remove expired entries
        pipe.zremrangebyscore(key, 0, window_start)
        # Count current entries
        pipe.zcard(key)
        # Add current request
        pipe.zadd(key, {str(now): now})
        # Set TTL on the key
        pipe.expire(key, self.window)
        results = await pipe.execute()

        request_count = results[1]

        if request_count >= self.limit:
            retry_after = int(self.window - (now - window_start))
            return JSONResponse(
                status_code=429,
                content={"detail": "Rate limit exceeded", "retry_after": retry_after},
                headers={
                    "Retry-After": str(retry_after),
                    "X-RateLimit-Limit": str(self.limit),
                    "X-RateLimit-Remaining": "0",
                }
            )

        response = await call_next(request)

        # Add rate limit headers to successful responses
        remaining = max(0, self.limit - request_count - 1)
        response.headers["X-RateLimit-Limit"] = str(self.limit)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"] = str(int(now + self.window))

        return response
```

Register it:

```python
# main.py
from fastapi import FastAPI
from middleware.rate_limiter import RateLimitMiddleware

app = FastAPI()

# Global rate limit: 100 requests per 60 seconds per IP per path
app.add_middleware(
    RateLimitMiddleware,
    redis_url="redis://localhost:6379/0",
    limit=100,
    window=60,
)
```

This gives you:
- **Sliding window** algorithm (accurate, no boundary burst problem)
- **Async Redis** (non-blocking, works with FastAPI's event loop)
- **Per-path limits** (each endpoint has its own counter)
- **Rate limit headers** on every response

---

### 2.3 Using FastAPI Dependencies — Lightweight Approach

For simple cases, you can use FastAPI's `Depends()` system without any extra packages:

```python
# dependencies/rate_limit.py
import time
from collections import defaultdict
from fastapi import Request, HTTPException


# In-memory store (use Redis in production)
_request_counts: dict[str, list[float]] = defaultdict(list)


def rate_limit(max_requests: int = 10, window_seconds: int = 60):
    """
    Returns a FastAPI dependency that enforces rate limiting.

    Usage:
        @app.get("/endpoint", dependencies=[Depends(rate_limit(10, 60))])
    """
    async def _rate_limit(request: Request):
        client_ip = request.client.host
        key = f"{client_ip}:{request.url.path}"
        now = time.time()

        # Remove timestamps outside the window
        _request_counts[key] = [
            ts for ts in _request_counts[key]
            if ts > now - window_seconds
        ]

        if len(_request_counts[key]) >= max_requests:
            raise HTTPException(
                status_code=429,
                detail=f"Rate limit exceeded. Try again in {window_seconds} seconds.",
                headers={"Retry-After": str(window_seconds)}
            )

        _request_counts[key].append(now)

    return _rate_limit
```

Use it:

```python
# main.py
from fastapi import FastAPI, Depends, Request
from dependencies.rate_limit import rate_limit

app = FastAPI()


@app.get("/api/products", dependencies=[Depends(rate_limit(60, 60))])
async def list_products():
    """60 requests per minute"""
    return {"products": []}


@app.post("/api/login", dependencies=[Depends(rate_limit(5, 60))])
async def login():
    """5 attempts per minute"""
    return {"message": "Login successful"}


@app.post("/api/payments", dependencies=[Depends(rate_limit(10, 60))])
async def process_payment():
    """10 payments per minute"""
    return {"payment": "processed"}
```

> **Limitation:** This in-memory approach only works for single-process deployments. For production with multiple Uvicorn workers behind Gunicorn, replace the `defaultdict` with Redis.

---

### 2.4 FastAPI Rate Limiting — Comparison Table

| Feature | SlowAPI | Custom Middleware | Depends() |
|---|---|---|---|
| **Extra packages** | `slowapi` | `redis` / `aioredis` | None |
| **Setup complexity** | Low | Medium | Low |
| **Per-endpoint limits** | Yes (decorator) | Manual (path-based) | Yes (per-dependency) |
| **Multiple limits per route** | Yes (stacked decorators) | Manual | Manual |
| **Dynamic limits (per-tier)** | Yes (callback function) | Yes (custom logic) | Yes (custom logic) |
| **Redis support** | Yes (`storage_uri`) | Yes (native) | Manual |
| **Rate limit headers** | Automatic | Manual (you add them) | Manual |
| **Async-native** | Partial (wraps `limits` lib) | Yes (fully async) | Yes |
| **Best for** | Most FastAPI projects | High-control, production | Quick prototypes, simple APIs |

---

### 2.5 Django vs FastAPI Rate Limiting — Head to Head

| Aspect | Django (DRF Throttling) | FastAPI (SlowAPI) |
|---|---|---|
| **Built-in?** | Yes (in DRF) | No (3rd party) |
| **Decorator syntax** | Class attribute (`throttle_classes`) | `@limiter.limit("60/minute")` |
| **Identifies client** | IP / User ID / Custom | IP / Header / Custom function |
| **Dynamic per-tier limits** | Override `get_rate()` method | Pass a callback function |
| **Cache backend** | Django cache framework | `limits` storage (Redis URI) |
| **Response code** | 429 + `Retry-After` | 429 + `Retry-After` |
| **Async support** | No (Django is sync by default) | Yes (FastAPI is async-first) |
| **Middleware alternative** | `django-ratelimit` decorators | Custom `BaseHTTPMiddleware` |
| **Multi-worker safety** | Redis required | Redis required |

The core concepts are the same. The main difference is that FastAPI gives you **async Redis** and **dependency injection** as first-class options, which are more natural for high-concurrency APIs.

---

## 3. Rate Limiting at the API Gateway Level

Now let's zoom out from Django and FastAPI.

You have grown. Your application is no longer a single Django or FastAPI monolith. It has been broken into **5 microservices**, all sitting behind an **API Gateway**.

### 3.1 The Architecture

Here are the 5 microservices:

| # | Service | Port | Responsibility |
|---|---|---|---|
| 1 | **User Service** | `8001` | Registration, profiles, authentication |
| 2 | **Product Service** | `8002` | Product catalog, search, details |
| 3 | **Order Service** | `8003` | Place orders, order history, tracking |
| 4 | **Payment Service** | `8004` | Process payments, refunds, invoices |
| 5 | **Notification Service** | `8005` | Email, SMS, push notifications |

All traffic enters through a single **API Gateway** (we will use **Kong** as the example, since it is the most widely used open-source gateway).

---

![Rate Limiting Architecture](/System_Design/images/rate_limiting_architecture.svg)

---

### 3.2 Why Rate Limit at the Gateway?

You might ask — *"Why not just add rate limiting inside each microservice?"*

You could. But here is why the gateway is better:

| Aspect | Rate Limit in Each Service | Rate Limit at Gateway |
|---|---|---|
| **Single point of control** | No — each service has its own config | Yes — one place to manage all rules |
| **Cross-service limits** | Impossible — services do not know about each other's traffic | Possible — gateway sees all traffic |
| **Bad traffic reaches services?** | Yes — request hits the service before being rejected | No — rejected at the edge |
| **Config consistency** | Hard to keep in sync | Centralized and consistent |
| **Resource waste** | Services waste CPU/memory processing bad requests | Bad requests never reach services |
| **Implementation effort** | 5× — implemented in every service | 1× — configured once at the gateway |

> **The bouncer analogy again:** It is more efficient to have one bouncer at the building entrance than one bouncer at every apartment door. The building bouncer rejects troublemakers before they even reach the hallway.

---

### 3.3 Rate Limiting Strategies at the Gateway

An API Gateway can enforce rate limits at multiple levels:

#### Level 1: Global Rate Limit Limit

A blanket limit on the entire gateway. No single client can overwhelm the system, regardless of which service they are calling.

```
Any client → API Gateway → Max 1000 requests/minute total
```

#### Level 2: Per-Service Rate Limit

Different services have different capacities. Your Payment Service processes slower than your Product Service because it talks to external payment providers.

```
Product Service  → 500 requests/minute per client
Order Service    → 200 requests/minute per client
Payment Service  →  50 requests/minute per client
Notification     → 100 requests/minute per client
User Service     → 150 requests/minute per client
```

#### Level 3: Per-Consumer Rate Limit

Different API consumers (tenants, plans, users) get different limits:

```
Free tier      →  100 requests/hour
Pro tier       → 5000 requests/hour
Enterprise     → 50000 requests/hour
Internal svc   → Unlimited
```

#### Level 4: Per-Route Rate Limit

Even within a service, some endpoints are heavier than others:

```
GET  /products       → 500/min  (read, cached, cheap)
POST /orders         → 100/min  (write, triggers payment)
POST /payments/charge →  20/min  (calls Stripe, expensive)
GET  /notifications  → 300/min  (read, lightweight)
```

---

### 3.4 Configuring Kong API Gateway — Full Setup

Let's configure Kong to rate-limit our 5 microservices. We will use Kong's declarative (YAML) configuration.

#### The Full Kong Configuration

```yaml
# kong.yml — Declarative Configuration
_format_version: "3.0"

# ─────────────────────────────────────────────
# 1. Define all 5 upstream services
# ─────────────────────────────────────────────
services:

  - name: user-service
    url: http://user-svc:8001
    routes:
      - name: user-routes
        paths:
          - /api/users
        strip_path: false

  - name: product-service
    url: http://product-svc:8002
    routes:
      - name: product-routes
        paths:
          - /api/products
        strip_path: false

  - name: order-service
    url: http://order-svc:8003
    routes:
      - name: order-routes
        paths:
          - /api/orders
        strip_path: false

  - name: payment-service
    url: http://payment-svc:8004
    routes:
      - name: payment-routes
        paths:
          - /api/payments
        strip_path: false

  - name: notification-service
    url: http://notification-svc:8005
    routes:
      - name: notification-routes
        paths:
          - /api/notifications
        strip_path: false

# ─────────────────────────────────────────────
# 2. Rate Limiting Plugins
# ─────────────────────────────────────────────
plugins:

  # ── GLOBAL RATE LIMIT ──
  # Applies to ALL services and routes.
  # No single IP can exceed 1000 requests per minute across all endpoints.
  - name: rate-limiting
    config:
      minute: 1000
      policy: redis
      redis:
        host: redis
        port: 6379
      limit_by: ip
      fault_tolerant: true      # If Redis is down, allow requests (fail-open)
      hide_client_headers: false # Show X-RateLimit headers in response

  # ── PER-SERVICE: Payment Service (strictest) ──
  # Payment operations are expensive — they call Stripe/PayPal.
  - name: rate-limiting
    service: payment-service
    config:
      minute: 50
      hour: 500
      policy: redis
      redis:
        host: redis
        port: 6379
      limit_by: consumer       # Rate limit per authenticated consumer
      fault_tolerant: false    # If Redis is down, BLOCK requests (fail-closed for payments)
      hide_client_headers: false

  # ── PER-SERVICE: Order Service ──
  - name: rate-limiting
    service: order-service
    config:
      minute: 200
      hour: 5000
      policy: redis
      redis:
        host: redis
        port: 6379
      limit_by: consumer
      fault_tolerant: true
      hide_client_headers: false

  # ── PER-SERVICE: Product Service (most relaxed) ──
  - name: rate-limiting
    service: product-service
    config:
      minute: 500
      hour: 20000
      policy: redis
      redis:
        host: redis
        port: 6379
      limit_by: ip
      fault_tolerant: true
      hide_client_headers: false

  # ── PER-SERVICE: User Service ──
  - name: rate-limiting
    service: user-service
    config:
      minute: 150
      hour: 3000
      policy: redis
      redis:
        host: redis
        port: 6379
      limit_by: consumer
      fault_tolerant: true
      hide_client_headers: false

  # ── PER-SERVICE: Notification Service ──
  - name: rate-limiting
    service: notification-service
    config:
      minute: 100
      hour: 2000
      policy: redis
      redis:
        host: redis
        port: 6379
      limit_by: consumer
      fault_tolerant: true
      hide_client_headers: false

# ─────────────────────────────────────────────
# 3. Consumer Tiers (for per-consumer limits)
# ─────────────────────────────────────────────
consumers:
  - username: free-tier-user
    plugins:
      - name: rate-limiting
        config:
          minute: 20
          hour: 100
          policy: redis
          redis:
            host: redis
            port: 6379

  - username: pro-tier-user
    plugins:
      - name: rate-limiting
        config:
          minute: 200
          hour: 5000
          policy: redis
          redis:
            host: redis
            port: 6379

  - username: enterprise-tier-user
    plugins:
      - name: rate-limiting
        config:
          minute: 2000
          hour: 50000
          policy: redis
          redis:
            host: redis
            port: 6379
```

#### Understanding the Plugin Precedence

Kong applies rate limiting plugins in this priority order:

```
┌─────────────────────────────────────────────────────────┐
│           Kong Rate Limit Plugin Precedence              │
│                                                         │
│   1. Consumer + Route   (most specific — wins)          │
│   2. Consumer + Service                                 │
│   3. Consumer           (per-consumer global)           │
│   4. Route              (per-route)                     │
│   5. Service            (per-service)                   │
│   6. Global             (least specific — fallback)     │
│                                                         │
│   More specific plugins OVERRIDE less specific ones.    │
└─────────────────────────────────────────────────────────┘
```

So if a pro-tier user hits the payment service:
1. Check: Is there a **Consumer + Route** plugin? → No
2. Check: Is there a **Consumer + Service** plugin? → No
3. Check: Is there a **Consumer** plugin? → Yes! `200/min, 5000/hr`
4. This **overrides** the global `1000/min` and the service-level `50/min`

---

### 3.5 What the Client Sees — Response Headers

Every rate-limited response from Kong includes these headers:

```
HTTP/1.1 200 OK
X-RateLimit-Limit-Minute: 200
X-RateLimit-Remaining-Minute: 187
X-RateLimit-Limit-Hour: 5000
X-RateLimit-Remaining-Hour: 4823
```

When the limit is exceeded:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 23
X-RateLimit-Limit-Minute: 200
X-RateLimit-Remaining-Minute: 0

{
    "message": "API rate limit exceeded"
}
```

---

### 3.6 Request Flow — Step by Step

Let's trace what happens when a **pro-tier user** sends a `POST /api/orders` request:

```
Step 1:  Client sends POST /api/orders
         Headers: Authorization: Bearer <jwt-token>
                  X-Consumer-Username: pro-tier-user

Step 2:  Kong receives the request at the edge

Step 3:  Authentication plugin runs first
         → Validates JWT token
         → Identifies consumer as "pro-tier-user"

Step 4:  Rate limiting plugin runs
         → Checks Redis: "How many requests has pro-tier-user
            made in the current minute?"
         → Redis returns: 45
         → Consumer limit is 200/min
         → 45 < 200 → ALLOW

Step 5:  Kong routes request to order-service:8003

Step 6:  Order Service processes the request
         → Returns 201 Created

Step 7:  Kong adds rate limit headers to the response
         → X-RateLimit-Remaining-Minute: 154

Step 8:  Client receives the response
```

---

![Rate Limiting Request Flow](/System_Design/images/rate_limiting_request_flow.svg)

---

### 3.7 Handling Rate Limit Exceeded — Client-Side Best Practice

When your client receives a `429`, it should not just crash. Here is a proper retry strategy:

```python
# client.py — Exponential Backoff with Retry-After
import requests
import time


def call_api_with_retry(url, headers, max_retries=3):
    """
    Calls an API with automatic retry on rate limiting.
    Respects the Retry-After header from the gateway.
    """
    for attempt in range(max_retries):
        response = requests.post(url, headers=headers)

        if response.status_code == 429:
            # Respect Retry-After header if present
            retry_after = int(response.headers.get('Retry-After', 2 ** attempt))
            print(f"Rate limited. Retrying in {retry_after}s (attempt {attempt + 1})")
            time.sleep(retry_after)
            continue

        return response

    raise Exception("Max retries exceeded — still rate limited")


# Usage
response = call_api_with_retry(
    url="https://api.example.com/api/orders",
    headers={"Authorization": "Bearer <token>"}
)
```

---

### 3.8 Advanced: Distributed Rate Limiting with Redis

In production, your API Gateway runs **multiple instances** behind a load balancer. The rate limit counter must be **shared** across all instances.

```
Client Request
      │
      ▼
┌──────────────┐
│ Load Balancer │
└──────┬───────┘
       │
   ┌───┴───┐
   │       │
   ▼       ▼
┌──────┐ ┌──────┐
│ Kong │ │ Kong │    ← Two gateway instances
│  #1  │ │  #2  │
└──┬───┘ └──┬───┘
   │        │
   └───┬────┘
       │
       ▼
   ┌───────┐
   │ Redis │    ← Shared counter store
   │       │
   │ Key: "ratelimit:pro-user:minute:202603041530"
   │ Value: 45
   │ TTL: 60 seconds
   └───────┘
```

Both Kong instances read/write the same Redis key. The counter is accurate regardless of which gateway instance handles the request.

**Redis Lua Script (Atomic Increment + Check):**

```lua
-- rate_limit.lua
-- Atomic rate limit check used by Kong internally
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('GET', key)

if current and tonumber(current) >= limit then
    return 0  -- Rate limit exceeded
end

current = redis.call('INCR', key)

if tonumber(current) == 1 then
    redis.call('EXPIRE', key, window)
end

return limit - tonumber(current)  -- Remaining requests
```

This Lua script runs **atomically** inside Redis — no race conditions between the read, check, and increment.

---

### 3.9 Rate Limiting at the Gateway — Other Popular Solutions

Kong is not the only option. Here is how other gateways handle rate limiting:

#### Nginx (using `limit_req`)

```nginx
# nginx.conf
http {
    # Define rate limit zones
    limit_req_zone $binary_remote_addr zone=global:10m rate=100r/s;
    limit_req_zone $binary_remote_addr zone=payments:10m rate=5r/s;
    limit_req_zone $binary_remote_addr zone=products:10m rate=50r/s;

    upstream user-service     { server user-svc:8001; }
    upstream product-service  { server product-svc:8002; }
    upstream order-service    { server order-svc:8003; }
    upstream payment-service  { server payment-svc:8004; }
    upstream notification-svc { server notification-svc:8005; }

    server {
        listen 80;

        # User Service
        location /api/users {
            limit_req zone=global burst=20 nodelay;
            proxy_pass http://user-service;
        }

        # Product Service — relaxed
        location /api/products {
            limit_req zone=products burst=100 nodelay;
            proxy_pass http://product-service;
        }

        # Order Service
        location /api/orders {
            limit_req zone=global burst=10 nodelay;
            proxy_pass http://order-service;
        }

        # Payment Service — strictest
        location /api/payments {
            limit_req zone=payments burst=5 nodelay;
            proxy_pass http://payment-service;
        }

        # Notification Service
        location /api/notifications {
            limit_req zone=global burst=15 nodelay;
            proxy_pass http://notification-svc;
        }
    }
}
```

> **`burst`** = how many extra requests to allow in a spike before rejecting.
> **`nodelay`** = process burst requests immediately rather than queuing them (uses leaky bucket without delay).

#### Traefik (using middleware)

```yaml
# traefik.yml
http:
  middlewares:
    rate-limit-global:
      rateLimit:
        average: 100    # requests per second
        burst: 50
        period: 1m

    rate-limit-payments:
      rateLimit:
        average: 5
        burst: 10
        period: 1m

  routers:
    products:
      rule: "PathPrefix(`/api/products`)"
      service: product-service
      middlewares:
        - rate-limit-global

    payments:
      rule: "PathPrefix(`/api/payments`)"
      service: payment-service
      middlewares:
        - rate-limit-payments
```

#### AWS API Gateway (using Usage Plans)

```json
{
    "usagePlan": {
        "name": "ProTier",
        "throttle": {
            "rateLimit": 200,
            "burstLimit": 50
        },
        "quota": {
            "limit": 50000,
            "period": "MONTH"
        },
        "apiStages": [
            {
                "apiId": "abc123",
                "stage": "prod",
                "throttle": {
                    "/api/payments/POST": {
                        "rateLimit": 10,
                        "burstLimit": 5
                    },
                    "/api/products/GET": {
                        "rateLimit": 500,
                        "burstLimit": 200
                    }
                }
            }
        ]
    }
}
```

#### Gateway Comparison for Rate Limiting

| Feature | Kong | Nginx | Traefik | AWS API Gateway |
|---|---|---|---|---|
| **Algorithm** | Sliding window (Redis) | Leaky bucket | Token bucket | Token bucket |
| **Per-consumer limits** | Yes (consumer groups) | Manual (via variables) | No (IP only) | Yes (usage plans + API keys) |
| **Per-route limits** | Yes | Yes (`location` blocks) | Yes (middleware per router) | Yes (method-level throttle) |
| **Distributed** | Yes (Redis-backed) | No (per-instance) | Yes (Redis with plugin) | Yes (managed by AWS) |
| **Dynamic config** | Yes (Admin API) | Requires reload | Yes (hot reload) | Yes (AWS API) |
| **Cost** | Free (OSS) / Paid (Enterprise) | Free | Free | Pay-per-request |

---

## 4. Rate Limiting Anti-Patterns

Here are mistakes teams commonly make:

| Anti-Pattern | Why It Fails | Better Approach |
|---|---|---|
| **Rate limit only by IP** | Multiple users behind one NAT share an IP. One user's abuse blocks everyone. | Combine IP + User ID + API key |
| **Same limit for all endpoints** | A product listing and a payment charge have very different costs | Use per-route limits based on endpoint cost |
| **Fail-open everywhere** | If Redis goes down, all limits are disabled — potential DDoS | Fail-closed for critical paths (payments), fail-open for reads |
| **No Retry-After header** | Clients hammer the server in a tight loop trying to get through | Always include `Retry-After` — clients can back off intelligently |
| **Hard-coded limits** | Changing limits requires a code deploy | Store limits in config/database, reload dynamically |
| **No monitoring on 429s** | You have no idea how many legitimate users are being rate limited | Track 429 rates per consumer/route — alert on spikes |

---

## 5. Production Checklist

Before going live with rate limiting, verify:

- [ ] **Redis is deployed** in a clustered/replicated setup (not single node)
- [ ] **Rate limit headers** are returned to clients (`X-RateLimit-*`)
- [ ] **Retry-After** header is included in 429 responses
- [ ] **Per-service limits** are tuned based on actual traffic patterns
- [ ] **Consumer tiers** are configured (free/pro/enterprise)
- [ ] **Fail-closed** is enabled for critical services (payments)
- [ ] **Fail-open** is enabled for non-critical services (product listing)
- [ ] **Monitoring dashboards** show 429 rates per service and per consumer
- [ ] **Alert rules** fire when 429 rate exceeds threshold
- [ ] **Load testing** has been done to validate limits under real traffic
- [ ] **Client SDKs** implement exponential backoff with jitter
- [ ] **Documentation** clearly states rate limits per tier

---

## 6. Decision Guide

```
What framework are you using?
├── Django (DRF) → Use DRF Throttling or django-ratelimit
│                   (simple, no infrastructure overhead)
│
├── FastAPI → Use SlowAPI for most projects
│             For full control → Custom Middleware + Redis
│             For prototypes → Depends() with in-memory store
│
└── Multiple Microservices?
    ├── Yes → Are they behind an API Gateway?
    │         ├── Yes → Configure rate limiting at the gateway
    │         │         (Kong / Nginx / Traefik / AWS API GW)
    │         │
    │         └── No → Add an API Gateway first!
    │                   Then configure rate limiting there.
    │
    └── No → You probably have a monolith.
              Use framework-level throttling (DRF / SlowAPI).
```

---

## 7. Summary

| Scope | Tool | Rate Limit Level |
|---|---|---|
| Single Django app | DRF Throttling / django-ratelimit | Per-view, per-user |
| Single FastAPI app | SlowAPI / Custom Middleware / Depends() | Per-endpoint, per-user, per-tier |
| Microservices + Gateway | Kong / Nginx / Traefik / AWS | Global, per-service, per-consumer, per-route |
| Distributed across instances | Redis-backed counters | Shared state, atomic operations |

Rate limiting is not just a "nice to have." It is a **core defense mechanism** that protects your infrastructure, ensures fair usage, and keeps your services alive under pressure. Start simple with Django or FastAPI throttling, and scale up to gateway-level rate limiting as your architecture grows.
