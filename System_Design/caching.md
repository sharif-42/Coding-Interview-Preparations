# Caching — From Basics to Production

You walk into a library every day and ask the librarian for the same book — *"Introduction to Algorithms"*.

Every single time, the librarian walks to the back, searches through thousands of shelves, finds the book, walks back, and hands it to you. Five minutes. Every time.

One day, the librarian gets smart. She keeps a copy of the book **on the front desk**. Now when you ask for it, she hands it to you in two seconds.

That front desk is a **cache**.

In software, caching means storing a copy of data in a **fast, temporary location** so that future requests for that data can be served faster — without going to the slow, original source (usually a database).

It is one of the most important concepts in system design. It reduces latency, lowers database load, saves money, and is the single easiest way to make a system faster.

It is also one of the most dangerous tools if misused. Stale data, inconsistency, memory leaks, and cache invalidation bugs have caused outages at some of the biggest tech companies in the world.

Let's go through everything — from how caching works, to Django and FastAPI implementations, to production-grade distributed caching with Redis.

---

## 1. Why Caching Matters

### Without a Cache

```
Client → API Server → Database
                      ↑
                  Every single request hits the DB.
                  10,000 users requesting the same product page?
                  That is 10,000 identical database queries.
```

### With a Cache

```
Client → API Server → Cache (Redis/Memcached)
                        │
                   Found in cache? ──── Yes → Return instantly (cache hit)
                        │
                        No (cache miss)
                        │
                        ▼
                     Database → Store result in cache → Return to client
```

### The Numbers That Matter

| Operation | Latency |
|---|---|
| Read from L1 CPU cache | ~1 nanosecond |
| Read from RAM | ~100 nanoseconds |
| Read from Redis (local) | ~0.5 milliseconds |
| Read from Redis (network) | ~1-2 milliseconds |
| Read from PostgreSQL (indexed) | ~5-50 milliseconds |
| Read from PostgreSQL (complex join) | ~50-500 milliseconds |
| Read from external API (Stripe, S3) | ~100-1000 milliseconds |

A cache hit from Redis is **10-100x faster** than a database query. For a system handling millions of requests, that difference is enormous.

---

![Caching Overview](/System_Design/images/caching_overview.svg)

---

## 2. Caching Strategies

Not all caching works the same way. There are five major strategies, each suited for different use cases.

### 2.1 Cache-Aside (Lazy Loading)

The most common pattern. The application manages the cache explicitly.

```
READ:
  1. Application checks cache for the key
  2. Cache hit?  → Return cached data
  3. Cache miss? → Query database → Store in cache → Return data

WRITE:
  1. Write directly to the database
  2. Invalidate (delete) the cache entry
  3. Next read will populate the cache again
```

```python
# Cache-Aside Example
import redis
import json

cache = redis.Redis(host='localhost', port=6379, db=0)

def get_product(product_id: int):
    cache_key = f"product:{product_id}"
    
    # Step 1: Check cache
    cached = cache.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache hit
    
    # Step 2: Cache miss — query database
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    
    # Step 3: Store in cache with TTL
    cache.setex(cache_key, 300, json.dumps(product))  # Cache for 5 minutes
    
    return product

def update_product(product_id: int, data: dict):
    # Step 1: Update database
    db.query("UPDATE products SET ... WHERE id = %s", product_id)
    
    # Step 2: Invalidate cache
    cache.delete(f"product:{product_id}")
```

**Pros:**
- Only caches data that is actually requested (no wasted memory)
- Cache failures do not break the system (fallback to DB)
- Simple to implement

**Cons:**
- First request is always slow (cache miss)
- Potential for stale data between write and invalidation

**Best for:** Read-heavy workloads, product catalogs, user profiles.

---

### 2.2 Read-Through

Similar to cache-aside, but the **cache itself** is responsible for loading data from the database. The application only talks to the cache.

```
READ:
  1. Application asks cache for data
  2. Cache hit?  → Return cached data
  3. Cache miss? → Cache queries the database itself → Stores it → Returns to app

The application never directly queries the database for reads.
```

**Difference from Cache-Aside:** In cache-aside, the *application* fetches from DB and populates the cache. In read-through, the *cache library* does it automatically.

**Pros:** Simpler application code — no cache miss logic.
**Cons:** Cache library must understand your data source. Less common in practice.

**Best for:** ORM-level caching, libraries like Hibernate's L2 cache.

---

### 2.3 Write-Through

Every write goes to the cache **and** the database at the same time. The cache is always consistent with the database.

```
WRITE:
  1. Application writes to cache
  2. Cache synchronously writes to database
  3. Both are updated atomically

READ:
  1. Always read from cache (it is always up-to-date)
```

```python
def update_product_write_through(product_id: int, data: dict):
    cache_key = f"product:{product_id}"
    
    # Step 1: Write to database
    db.query("UPDATE products SET ... WHERE id = %s", product_id)
    
    # Step 2: Write to cache (not delete — update it)
    cache.setex(cache_key, 300, json.dumps(data))
```

**Pros:** Cache is never stale — reads are always consistent.
**Cons:** Every write is slower (double write — DB + cache). Wastes cache memory on data that may never be read.

**Best for:** Systems where consistency is critical and reads are very frequent.

---

### 2.4 Write-Behind (Write-Back)

Writes go to the cache first. The cache asynchronously flushes to the database later.

```
WRITE:
  1. Application writes to cache
  2. Cache acknowledges immediately (fast!)
  3. Cache asynchronously writes to database (batched, delayed)

READ:
  1. Read from cache (always has latest data)
```

**Pros:** Extremely fast writes. Can batch multiple writes into a single DB operation.
**Cons:** Risk of data loss if the cache crashes before flushing to DB. Complex to implement correctly.

**Best for:** Write-heavy workloads (analytics, logging, metrics), where losing a few seconds of data is acceptable.

---

### 2.5 Write-Around

Writes go directly to the database, bypassing the cache. The cache only gets populated on reads (like cache-aside).

```
WRITE:
  1. Write directly to database (skip cache entirely)

READ:
  1. Check cache → Miss → Query DB → Populate cache
```

**Pros:** Cache is not polluted with data that may never be read.
**Cons:** First read after a write is always a cache miss.

**Best for:** Systems where written data is rarely read immediately (log entries, audit trails).

---

### Strategy Comparison

| Strategy | Read Performance | Write Performance | Consistency | Complexity | Data Loss Risk |
|---|---|---|---|---|---|
| **Cache-Aside** | Fast (after first miss) | Normal (DB write + invalidate) | Eventual | Low | None |
| **Read-Through** | Fast (after first miss) | Normal | Eventual | Medium | None |
| **Write-Through** | Always fast | Slow (double write) | Strong | Medium | None |
| **Write-Behind** | Always fast | Very fast | Eventual | High | Yes (cache crash) |
| **Write-Around** | Slow on first read | Normal (DB only) | Strong (DB is source) | Low | None |

> **What most production systems use:** **Cache-Aside** for reads + **invalidation on write**. It is the simplest, most reliable, and covers 90% of use cases.

---

![Caching Strategies](/System_Design/images/caching_strategies.svg)

---

## 3. Cache Invalidation — The Hard Problem

> *"There are only two hard things in Computer Science: cache invalidation and naming things."*
> — Phil Karlton

Cache invalidation is deciding **when to remove or update cached data** so that clients never see stale (outdated) information.

### 3.1 TTL (Time-To-Live)

The simplest approach. Every cache entry has an expiration time. After the TTL expires, the entry is automatically deleted.

```python
# Set with a 5-minute TTL
cache.setex("product:123", 300, json.dumps(product_data))

# After 300 seconds, Redis automatically deletes this key
```

**TTL Guidelines:**

| Data Type | Suggested TTL | Reasoning |
|---|---|---|
| User session | 30 minutes – 24 hours | Balance security with UX |
| Product listing | 5 – 15 minutes | Products do not change every second |
| Search results | 1 – 5 minutes | Results can be slightly stale |
| User profile | 5 – 30 minutes | Profiles change infrequently |
| Stock prices / live data | 1 – 10 seconds | Must be near real-time |
| Static config (feature flags) | 1 – 5 minutes | Changes are rare but must propagate |
| Homepage banner | 15 – 60 minutes | Changes on a schedule |

**Problem with TTL alone:** For 5 minutes, clients might see stale data. If a product price changes, a user could see the old price for up to 5 minutes.

---

### 3.2 Event-Based Invalidation

When data changes, explicitly invalidate the cache. This is more precise than TTL.

```python
def update_product(product_id: int, data: dict):
    # Update database
    db.query("UPDATE products SET price = %s WHERE id = %s", data['price'], product_id)
    
    # Immediately invalidate cache
    cache.delete(f"product:{product_id}")
    
    # Also invalidate related caches
    cache.delete(f"product_list:category:{data['category_id']}")
    cache.delete("homepage:featured_products")
```

**Problem:** You need to know **every cache key** that depends on the changed data. Miss one, and you have stale data somewhere.

---

### 3.3 Version-Based Invalidation

Instead of deleting cache entries, include a version number in the key. When data changes, increment the version — old keys naturally become orphaned.

```python
def get_product(product_id: int):
    # Get current version from a cheap source (e.g., a Redis counter)
    version = cache.get(f"product_version:{product_id}") or "1"
    
    cache_key = f"product:{product_id}:v{version}"
    cached = cache.get(cache_key)
    
    if cached:
        return json.loads(cached)
    
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    cache.setex(cache_key, 300, json.dumps(product))
    return product

def update_product(product_id: int, data: dict):
    db.query("UPDATE products SET ... WHERE id = %s", product_id)
    
    # Increment version — old cache keys are now stale (will be evicted by TTL)
    cache.incr(f"product_version:{product_id}")
```

---

### 3.4 Combined Strategy (Recommended for Production)

Use **TTL as a safety net** + **event-based invalidation for immediacy**:

```python
def update_product(product_id: int, data: dict):
    # 1. Write to database
    db.query("UPDATE products SET price = %s WHERE id = %s", data['price'], product_id)
    
    # 2. Immediately invalidate (event-based)
    cache.delete(f"product:{product_id}")
    
    # Even if the invalidation message is lost (network issue, bug),
    # the TTL ensures the stale cache entry expires within 5 minutes.
```

This gives you:
- **Immediate consistency** in the happy path (cache is deleted instantly)
- **Bounded staleness** in the failure path (TTL guarantees eventual consistency)

---

## 4. Caching in Django

Django has a powerful, built-in cache framework that supports multiple backends and caching levels.

### 4.1 Cache Backend Configuration

```python
# settings.py

# ── Option 1: Redis (recommended for production) ──
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SERIALIZER': 'django_redis.serializers.json.JSONSerializer',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'RETRY_ON_TIMEOUT': True,
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,  # Default TTL: 5 minutes
    }
}

# ── Option 2: Memcached ──
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}

# ── Option 3: Local memory (development only) ──
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake',
    }
}

# ── Option 4: File-based (rarely used) ──
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/tmp/django_cache',
    }
}
```

### 4.2 Per-View Caching

The simplest way to cache in Django — cache the entire response of a view.

```python
# views.py
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def product_list(request):
    """
    The entire response is cached.
    For 15 minutes, Django serves the cached HTML/JSON
    without executing this function at all.
    """
    products = Product.objects.all()
    return render(request, 'products/list.html', {'products': products})
```

For class-based views:

```python
# urls.py
from django.views.decorators.cache import cache_page
from .views import ProductListView

urlpatterns = [
    path('products/', cache_page(60 * 15)(ProductListView.as_view())),
]
```

**With `Vary` header** — cache different versions based on headers:

```python
from django.views.decorators.vary import vary_on_headers

@vary_on_headers('Authorization')
@cache_page(60 * 15)
def user_dashboard(request):
    """
    Different cache entry per Authorization header.
    User A and User B get separate cached responses.
    """
    ...
```

### 4.3 Template Fragment Caching

Cache individual parts of a template instead of the whole page:

```html
{% load cache %}

<!-- Cache the sidebar for 10 minutes -->
{% cache 600 sidebar %}
    <div class="sidebar">
        {% for category in categories %}
            <a href="{{ category.url }}">{{ category.name }}</a>
        {% endfor %}
    </div>
{% endcache %}

<!-- Cache per-user navigation -->
{% cache 600 user_nav request.user.id %}
    <nav>Welcome, {{ request.user.username }}</nav>
{% endcache %}

<!-- Uncached dynamic content -->
<div class="main-content">
    {{ dynamic_content }}
</div>
```

The second argument is the cache fragment name, and additional arguments create unique cache keys (e.g., per-user).

### 4.4 Low-Level Cache API

For maximum control, use Django's cache API directly:

```python
from django.core.cache import cache

# ── Basic Operations ──
cache.set('my_key', 'my_value', timeout=300)    # Set with 5-min TTL
value = cache.get('my_key')                       # Get (returns None on miss)
value = cache.get('my_key', 'default_value')      # Get with default
cache.delete('my_key')                            # Delete

# ── Atomic Operations ──
cache.set('counter', 0)
cache.incr('counter')       # Increment by 1
cache.incr('counter', 10)   # Increment by 10
cache.decr('counter')       # Decrement by 1

# ── Bulk Operations ──
cache.set_many({'key1': 'val1', 'key2': 'val2'}, timeout=300)
values = cache.get_many(['key1', 'key2'])   # Returns dict: {'key1': 'val1', 'key2': 'val2'}
cache.delete_many(['key1', 'key2'])

# ── Get or Set (atomic) ──
# If key exists → return cached value
# If key missing → call the function, cache result, return it
value = cache.get_or_set('expensive_query', lambda: Product.objects.count(), timeout=300)

# ── Check Existence ──
cache.has_key('my_key')   # True/False (doesn't load value into memory)

# ── Clear All ──
cache.clear()   # Deletes everything — use with caution!
```

### 4.5 Cache-Aside Pattern in DRF

Here is a real production example for a Django REST Framework API:

```python
# views.py
from django.core.cache import cache
from rest_framework.views import APIView
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer


class ProductDetailView(APIView):
    def get(self, request, product_id):
        cache_key = f"product:{product_id}"
        
        # Try cache first
        product_data = cache.get(cache_key)
        
        if product_data is None:
            # Cache miss — hit database
            try:
                product = Product.objects.select_related('category').get(id=product_id)
            except Product.DoesNotExist:
                return Response({"error": "Product not found"}, status=404)
            
            serializer = ProductSerializer(product)
            product_data = serializer.data
            
            # Store in cache for 10 minutes
            cache.set(cache_key, product_data, timeout=600)
        
        return Response(product_data)


class ProductUpdateView(APIView):
    def put(self, request, product_id):
        product = Product.objects.get(id=product_id)
        serializer = ProductSerializer(product, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        
        # Invalidate cache
        cache.delete(f"product:{product_id}")
        
        # Also invalidate list caches that include this product
        cache.delete(f"product_list:category:{product.category_id}")
        cache.delete("product_list:featured")
        
        return Response(serializer.data)
```

### 4.6 QuerySet Caching with Decorators

Build a reusable caching decorator for views:

```python
# decorators.py
import functools
import hashlib
from django.core.cache import cache


def cached_view(timeout=300, key_prefix='view'):
    """
    Decorator that caches the view response.
    Cache key is based on the request path and query parameters.
    """
    def decorator(view_func):
        @functools.wraps(view_func)
        def wrapper(request, *args, **kwargs):
            # Build a unique cache key from the request
            key_data = f"{key_prefix}:{request.path}:{request.GET.urlencode()}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()
            
            response = cache.get(cache_key)
            if response is None:
                response = view_func(request, *args, **kwargs)
                cache.set(cache_key, response, timeout=timeout)
            
            return response
        return wrapper
    return decorator


# Usage
@cached_view(timeout=600, key_prefix='products')
def product_list(request):
    products = Product.objects.filter(is_active=True)
    return JsonResponse({"products": list(products.values())})
```

---

## 5. Caching in FastAPI

FastAPI does not have a built-in cache framework like Django. But its async nature makes it ideal for high-performance caching with Redis.

### 5.1 Using `fastapi-cache2` — The Popular Way

```bash
pip install fastapi-cache2[redis]
```

```python
# main.py
from fastapi import FastAPI
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache
from redis import asyncio as aioredis

app = FastAPI()


@app.on_event("startup")
async def startup():
    redis = aioredis.from_url("redis://localhost:6379/0")
    FastAPICache.init(RedisBackend(redis), prefix="myapp")


@app.get("/api/products")
@cache(expire=300)  # Cache for 5 minutes
async def list_products():
    """
    First request: hits the database, caches result.
    Subsequent requests: served from Redis instantly.
    """
    products = await db.fetch_all("SELECT * FROM products WHERE is_active = true")
    return {"products": products}


@app.get("/api/products/{product_id}")
@cache(expire=600)  # Cache for 10 minutes
async def get_product(product_id: int):
    product = await db.fetch_one("SELECT * FROM products WHERE id = :id", {"id": product_id})
    if not product:
        raise HTTPException(status_code=404, detail="Not found")
    return {"product": dict(product)}
```

The `@cache` decorator automatically:
- Generates a cache key from the function name + arguments
- Checks Redis on each request
- Stores the response on cache miss
- Returns the cached response on cache hit

### 5.2 Manual Cache-Aside with Redis

For full control, use Redis directly:

```python
# main.py
from fastapi import FastAPI, HTTPException
from redis import asyncio as aioredis
import json

app = FastAPI()
redis = aioredis.from_url("redis://localhost:6379/0", decode_responses=True)


@app.get("/api/products/{product_id}")
async def get_product(product_id: int):
    cache_key = f"product:{product_id}"
    
    # Step 1: Check cache
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Step 2: Cache miss — query database
    product = await db.fetch_one(
        "SELECT * FROM products WHERE id = :id",
        {"id": product_id}
    )
    
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    
    product_data = dict(product)
    
    # Step 3: Store in cache
    await redis.setex(cache_key, 600, json.dumps(product_data))
    
    return product_data


@app.put("/api/products/{product_id}")
async def update_product(product_id: int, data: dict):
    # Update database
    await db.execute(
        "UPDATE products SET name = :name, price = :price WHERE id = :id",
        {**data, "id": product_id}
    )
    
    # Invalidate cache
    await redis.delete(f"product:{product_id}")
    
    return {"message": "Updated", "product_id": product_id}
```

### 5.3 Caching with Dependencies

Use FastAPI's dependency injection for cleaner cache logic:

```python
# dependencies/cache.py
from fastapi import Request
from redis import asyncio as aioredis
import json

redis = aioredis.from_url("redis://localhost:6379/0", decode_responses=True)


class CacheService:
    """
    Reusable cache service injected via FastAPI's Depends().
    """
    
    def __init__(self, prefix: str = "app"):
        self.prefix = prefix
    
    def _key(self, key: str) -> str:
        return f"{self.prefix}:{key}"
    
    async def get(self, key: str):
        data = await redis.get(self._key(key))
        return json.loads(data) if data else None
    
    async def set(self, key: str, value, ttl: int = 300):
        await redis.setex(self._key(key), ttl, json.dumps(value))
    
    async def delete(self, key: str):
        await redis.delete(self._key(key))
    
    async def delete_pattern(self, pattern: str):
        """Delete all keys matching a pattern (e.g., 'product:*')."""
        keys = []
        async for key in redis.scan_iter(match=self._key(pattern)):
            keys.append(key)
        if keys:
            await redis.delete(*keys)


# Usage in views
from fastapi import FastAPI, Depends

app = FastAPI()
cache_service = CacheService(prefix="myapp")


@app.get("/api/products/{product_id}")
async def get_product(product_id: int):
    cached = await cache_service.get(f"product:{product_id}")
    if cached:
        return cached
    
    product = await fetch_product_from_db(product_id)
    await cache_service.set(f"product:{product_id}", product, ttl=600)
    return product


@app.put("/api/products/{product_id}")
async def update_product(product_id: int, data: dict):
    await update_product_in_db(product_id, data)
    await cache_service.delete(f"product:{product_id}")
    await cache_service.delete_pattern("product_list:*")
    return {"message": "Updated"}
```

### 5.4 Response Caching with Middleware

For caching entire responses at the middleware level:

```python
# middleware/cache.py
import hashlib
import json
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse
from redis import asyncio as aioredis
from fastapi import Request

redis = aioredis.from_url("redis://localhost:6379/0", decode_responses=True)


class ResponseCacheMiddleware(BaseHTTPMiddleware):
    """
    Cache GET responses at the middleware level.
    Similar to Django's per-view caching.
    """

    def __init__(self, app, ttl: int = 300):
        super().__init__(app)
        self.ttl = ttl

    async def dispatch(self, request: Request, call_next):
        # Only cache GET requests
        if request.method != "GET":
            return await call_next(request)

        # Build cache key from method + path + query params
        key_source = f"{request.method}:{request.url.path}:{request.url.query}"
        cache_key = f"response:{hashlib.md5(key_source.encode()).hexdigest()}"

        # Check cache
        cached = await redis.get(cache_key)
        if cached:
            data = json.loads(cached)
            return JSONResponse(
                content=data["body"],
                status_code=data["status"],
                headers={"X-Cache": "HIT"},
            )

        # Cache miss — process request
        response = await call_next(request)

        # Only cache successful responses
        if 200 <= response.status_code < 300:
            body = b""
            async for chunk in response.body_iterator:
                body += chunk

            await redis.setex(
                cache_key,
                self.ttl,
                json.dumps({"body": json.loads(body), "status": response.status_code}),
            )

            return JSONResponse(
                content=json.loads(body),
                status_code=response.status_code,
                headers={"X-Cache": "MISS"},
            )

        return response
```

Register it:

```python
app.add_middleware(ResponseCacheMiddleware, ttl=300)
```

---

## 6. Django vs FastAPI Caching — Comparison

| Aspect | Django | FastAPI |
|---|---|---|
| **Built-in cache framework** | Yes (`django.core.cache`) | No (use `fastapi-cache2` or manual) |
| **Backend support** | Redis, Memcached, file, local memory | Redis (via `aioredis`), in-memory |
| **Per-view caching** | `@cache_page(timeout)` decorator | `@cache(expire=timeout)` with fastapi-cache2 |
| **Template fragment caching** | `{% cache %}` template tag | N/A (FastAPI does not render templates typically) |
| **Low-level API** | `cache.get()`, `cache.set()`, `cache.delete()` | `await redis.get()`, `await redis.setex()` |
| **Async support** | Limited (Django 4.1+ has some) | Fully async (native `await`) |
| **Cache key generation** | Automatic (URL-based) or manual | Manual (or automatic with fastapi-cache2) |
| **Multiple cache backends** | Yes (define multiple in settings) | Manual (create multiple Redis connections) |
| **ORM-level caching** | django-cacheops, django-cachalot | Manual |
| **Middleware caching** | `UpdateCacheMiddleware` + `FetchFromCacheMiddleware` | Custom middleware (as shown above) |

---

## 7. Redis Deep Dive — The Caching Engine

Redis is the most popular caching backend. Understanding it well is critical for senior interviews.

### 7.1 Why Redis?

| Feature | Why It Matters |
|---|---|
| **In-memory** | All data in RAM — microsecond reads |
| **Single-threaded** | No race conditions, atomic operations by design |
| **Rich data structures** | Not just key-value — lists, sets, sorted sets, hashes, streams |
| **Persistence** | Optional RDB snapshots + AOF logs — data survives restarts |
| **Pub/Sub** | Built-in messaging for cache invalidation across services |
| **Lua scripting** | Run atomic multi-step operations server-side |
| **Cluster mode** | Scale horizontally across multiple nodes |
| **TTL built-in** | Automatic expiration — no cleanup code needed |

### 7.2 Data Structures for Caching

```
# String — most common, simple key-value
SET product:123 '{"name": "Widget", "price": 29.99}'
GET product:123

# Hash — store objects without serialization
HSET product:123 name "Widget" price 29.99 stock 150
HGET product:123 price
HGETALL product:123

# List — recent items, activity feeds
LPUSH user:456:recent_views "product:123" "product:789"
LRANGE user:456:recent_views 0 9    # Last 10 viewed products

# Set — unique collections, tags
SADD product:123:tags "electronics" "sale" "featured"
SMEMBERS product:123:tags
SINTER product:123:tags product:456:tags   # Common tags

# Sorted Set — leaderboards, ranked results
ZADD trending_products 1500 "product:123" 1200 "product:456"
ZREVRANGE trending_products 0 9   # Top 10 by score
```

### 7.3 Eviction Policies

When Redis runs out of memory, it needs to decide which keys to remove. This is configured via `maxmemory-policy`:

| Policy | Behavior | Best For |
|---|---|---|
| `noeviction` | Return errors when memory is full | When you cannot afford to lose any key |
| `allkeys-lru` | Remove least recently used key (any key) | **General-purpose caching (recommended)** |
| `volatile-lru` | Remove least recently used key (only keys with TTL) | Mix of cache + permanent data |
| `allkeys-lfu` | Remove least frequently used key | When access frequency matters more than recency |
| `volatile-lfu` | LFU on keys with TTL | Frequency-based, TTL keys only |
| `allkeys-random` | Remove a random key | When all keys have equal importance |
| `volatile-random` | Remove a random key with TTL | Random eviction from TTL keys |
| `volatile-ttl` | Remove keys closest to expiration | Prioritize keeping keys with longer TTL |

```
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

> **Interview tip:** `allkeys-lru` is the best default for caching. It evicts the least recently used key regardless of whether it has a TTL. This ensures your most active data stays in memory.

### 7.4 Persistence Options

| Option | How It Works | Data Loss | Performance Impact |
|---|---|---|---|
| **No persistence** | Pure in-memory cache | All data on restart | None (fastest) |
| **RDB (snapshots)** | Periodic point-in-time snapshots to disk | Up to last snapshot interval | Minimal |
| **AOF (append-only file)** | Logs every write operation | Depends on `fsync` policy | Moderate |
| **RDB + AOF** | Both — AOF for durability, RDB for backups | Minimal | Moderate |

For caching, **no persistence or RDB-only** is sufficient. The database is the source of truth — the cache can be rebuilt.

---

## 8. Caching Layers — Where to Cache

A production system caches at multiple levels:

```
Client (Browser)
    │  ← Browser cache, localStorage, Service Worker
    ▼
CDN (CloudFront, Cloudflare)
    │  ← Static assets, API responses with Cache-Control
    ▼
API Gateway (Kong, Nginx)
    │  ← Full response caching, rate limit counters
    ▼
Application (Django/FastAPI)
    │  ← Redis/Memcached — business logic results
    ▼
Database
    │  ← Query cache, materialized views, indexes
    ▼
Disk
```

### Layer-by-Layer Breakdown

| Layer | What to Cache | TTL | Tool |
|---|---|---|---|
| **Browser** | Static assets (JS, CSS, images), API responses | Hours to days | `Cache-Control`, `ETag` headers |
| **CDN** | Images, videos, static pages, some API responses | Minutes to days | CloudFront, Cloudflare, Fastly |
| **API Gateway** | Full API responses, authentication results | Seconds to minutes | Kong proxy-cache plugin, Nginx `proxy_cache` |
| **Application** | DB query results, computed values, session data | Seconds to minutes | Redis, Memcached |
| **Database** | Query plans, frequently-accessed rows | Automatic | PostgreSQL query cache, MySQL query cache |

### HTTP Cache Headers (Browser + CDN)

```python
# FastAPI — set cache headers on response
from fastapi import FastAPI
from fastapi.responses import JSONResponse

@app.get("/api/products")
async def list_products():
    products = await get_products()
    return JSONResponse(
        content={"products": products},
        headers={
            "Cache-Control": "public, max-age=300, s-maxage=600",
            # public    = CDN can cache this
            # max-age   = Browser caches for 5 minutes
            # s-maxage  = CDN caches for 10 minutes
        }
    )

@app.get("/api/user/profile")
async def get_profile():
    profile = await get_user_profile()
    return JSONResponse(
        content=profile,
        headers={
            "Cache-Control": "private, max-age=60",
            # private  = Only browser can cache (NOT CDN) — user-specific data
            # max-age  = Browser caches for 1 minute
        }
    )

@app.post("/api/orders")
async def create_order():
    # Write operations should never be cached
    ...
    return JSONResponse(
        content={"order_id": "ORD-123"},
        headers={
            "Cache-Control": "no-store",
            # no-store = Do not cache anywhere
        }
    )
```

```python
# Django — set cache headers
from django.views.decorators.cache import cache_control

@cache_control(public=True, max_age=300, s_maxage=600)
def product_list(request):
    ...

@cache_control(private=True, max_age=60)
def user_profile(request):
    ...

@cache_control(no_store=True)
def create_order(request):
    ...
```

---

![Caching Layers](/System_Design/images/caching_layers.svg)

---

## 9. Cache Problems and Solutions

### 9.1 Cache Stampede (Thundering Herd)

**Problem:** A popular cache key expires. Hundreds of requests arrive simultaneously, all hit the database at the same time (cache miss). The database gets overwhelmed.

```
Cache key "popular_product" expires at 10:00:00
  10:00:01 → Request 1: cache miss → hits DB
  10:00:01 → Request 2: cache miss → hits DB
  10:00:01 → Request 3: cache miss → hits DB
  ...
  10:00:01 → Request 500: cache miss → hits DB
  
  Database: 500 identical queries at once = potential crash
```

**Solution: Locking (Mutex)**

```python
import time

async def get_popular_product(product_id: int):
    cache_key = f"product:{product_id}"
    lock_key = f"lock:{cache_key}"
    
    # Try cache
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Try to acquire lock (only one request rebuilds the cache)
    lock_acquired = await redis.set(lock_key, "1", ex=10, nx=True)
    
    if lock_acquired:
        # This request rebuilds the cache
        product = await fetch_from_db(product_id)
        await redis.setex(cache_key, 300, json.dumps(product))
        await redis.delete(lock_key)
        return product
    else:
        # Another request is rebuilding — wait and retry
        await asyncio.sleep(0.1)
        return await get_popular_product(product_id)  # Retry
```

**Solution: Early Refresh**

Refresh the cache **before** it expires:

```python
async def get_product_with_early_refresh(product_id: int):
    cache_key = f"product:{product_id}"
    
    cached = await redis.get(cache_key)
    if cached:
        data = json.loads(cached)
        ttl = await redis.ttl(cache_key)
        
        # If TTL is less than 20% remaining, refresh in background
        if ttl < 60:  # Less than 1 minute left (out of 5 min TTL)
            asyncio.create_task(refresh_cache(product_id))
        
        return data
    
    # Full cache miss
    return await rebuild_and_cache(product_id)
```

---

### 9.2 Cache Penetration

**Problem:** Requests for data that **does not exist** in the database. Every request is a cache miss → hits the DB → finds nothing → does not cache anything → next request repeats.

```
GET /api/products/999999  (product does not exist)
  → Cache miss → DB query → No result → Nothing cached
  
Attacker sends 10,000 requests for /api/products/999999
  → 10,000 database queries for a non-existent product
```

**Solution: Cache null results**

```python
async def get_product(product_id: int):
    cache_key = f"product:{product_id}"
    
    cached = await redis.get(cache_key)
    if cached == "NULL":
        return None  # Known non-existent — skip DB
    if cached:
        return json.loads(cached)
    
    product = await fetch_from_db(product_id)
    
    if product is None:
        # Cache the null result with a short TTL
        await redis.setex(cache_key, 60, "NULL")  # 1 minute
        return None
    
    await redis.setex(cache_key, 300, json.dumps(product))
    return product
```

**Solution: Bloom Filter**

For large-scale systems, use a Bloom filter to check if a key *could* exist before hitting the cache or DB:

```python
# A Bloom filter tells you:
# - "Definitely NOT in set" → skip DB entirely
# - "Probably in set" → proceed with cache/DB lookup

# Redis has a Bloom filter module (RedisBloom)
# BF.ADD products 123    (add product ID)
# BF.EXISTS products 999999  → 0 (definitely not in set)
```

---

### 9.3 Cache Avalanche

**Problem:** Many cache keys expire at the **same time** (e.g., all set with the same TTL at startup). Massive cache miss storm hits the database.

**Solution: Add jitter to TTL**

```python
import random

def cache_with_jitter(key: str, value, base_ttl: int = 300):
    """
    Add random jitter (±20%) to prevent mass expiration.
    base_ttl=300 → actual TTL between 240 and 360 seconds.
    """
    jitter = random.randint(-base_ttl // 5, base_ttl // 5)
    actual_ttl = base_ttl + jitter
    cache.setex(key, actual_ttl, json.dumps(value))
```

**Solution: Staggered warm-up**

When deploying or restarting, warm up the cache gradually instead of all at once.

---

### 9.4 Hot Key Problem

**Problem:** One key is so popular that it overwhelms a single Redis node. Think: the Super Bowl homepage, a viral product, breaking news.

**Solution: Local caching + Redis**

```python
from functools import lru_cache
import time

# L1: In-process memory cache (per-worker)
_local_cache = {}

async def get_hot_product(product_id: int):
    cache_key = f"product:{product_id}"
    
    # L1: Check local memory (microseconds)
    if cache_key in _local_cache:
        entry = _local_cache[cache_key]
        if time.time() - entry['time'] < 5:  # 5-second local TTL
            return entry['data']
    
    # L2: Check Redis (milliseconds)
    cached = await redis.get(cache_key)
    if cached:
        data = json.loads(cached)
        _local_cache[cache_key] = {'data': data, 'time': time.time()}
        return data
    
    # L3: Database (tens of milliseconds)
    data = await fetch_from_db(product_id)
    await redis.setex(cache_key, 300, json.dumps(data))
    _local_cache[cache_key] = {'data': data, 'time': time.time()}
    return data
```

**Solution: Read replicas**

Add Redis read replicas to distribute the load for hot keys across multiple nodes.

---

### Problem Summary

| Problem | What Happens | Solution |
|---|---|---|
| **Cache Stampede** | Popular key expires → hundreds of DB queries at once | Locking (mutex) or early refresh |
| **Cache Penetration** | Repeated queries for non-existent data → DB hammered | Cache null results + Bloom filter |
| **Cache Avalanche** | Mass key expiration → DB overwhelmed | TTL jitter + staggered warm-up |
| **Hot Key** | One key gets too many requests → single Redis node overloaded | Local cache (L1) + read replicas |

---

## 10. Caching at the API Gateway

If you are running microservices behind an API Gateway, the gateway can cache responses to reduce load on all downstream services.

### Kong Proxy Cache Plugin

```yaml
# kong.yml
plugins:
  - name: proxy-cache
    config:
      response_code:
        - 200                    # Only cache 200 OK
      request_method:
        - GET                    # Only cache GET requests
        - HEAD
      content_type:
        - application/json       # Only cache JSON responses
      cache_ttl: 300             # 5 minutes
      strategy: memory           # or "redis" for distributed
      memory:
        dictionary_name: kong_cache
```

### Nginx Proxy Cache

```nginx
http {
    # Define cache zone
    proxy_cache_path /tmp/nginx_cache levels=1:2
                     keys_zone=api_cache:10m
                     max_size=1g
                     inactive=10m;

    server {
        location /api/products {
            proxy_cache api_cache;
            proxy_cache_valid 200 5m;         # Cache 200s for 5 minutes
            proxy_cache_valid 404 1m;         # Cache 404s for 1 minute
            proxy_cache_key "$scheme$request_method$host$request_uri";
            
            add_header X-Cache-Status $upstream_cache_status;  # HIT/MISS
            
            proxy_pass http://product-service;
        }

        # Never cache write operations
        location /api/orders {
            proxy_cache off;
            proxy_pass http://order-service;
        }
    }
}
```

---

## 11. Production Checklist

Before deploying caching in production:

- [ ] **Redis is deployed** in a replicated setup (at minimum primary + replica)
- [ ] **Eviction policy** is set to `allkeys-lru` (or appropriate for your use case)
- [ ] **Max memory** is configured (`maxmemory 2gb`) — do not let Redis consume all RAM
- [ ] **TTL is set** on every cache key — no keys should live forever unless intentional
- [ ] **TTL jitter** is applied to prevent cache avalanche
- [ ] **Cache-aside with invalidation** is used for read-write data
- [ ] **Null results are cached** to prevent cache penetration
- [ ] **Stampede protection** (locking or early refresh) is in place for hot keys
- [ ] **Cache hit/miss ratio** is monitored — target >90% hit rate
- [ ] **Memory usage** is monitored with alerts before hitting maxmemory
- [ ] **Serialization** is efficient (JSON or MessagePack, not pickle)
- [ ] **Key naming convention** is consistent (`service:entity:id` pattern)
- [ ] **Cache warming** strategy exists for cold starts / deployments
- [ ] **X-Cache header** is returned in responses (HIT/MISS) for debugging

---

## 12. Decision Guide

```
Is your data read-heavy and changes infrequently?
├── Yes → Cache it!
│   │
│   ├── Single Django/FastAPI app?
│   │   ├── Django → Use built-in cache framework + Redis
│   │   └── FastAPI → Use fastapi-cache2 or manual Redis
│   │
│   ├── Behind an API Gateway?
│   │   └── Add proxy-cache at the gateway level (Kong/Nginx)
│   │       + application-level Redis cache
│   │
│   └── Need sub-millisecond reads?
│       └── Add in-process local cache (LRU) as L1
│           + Redis as L2
│           + Database as L3
│
├── Data changes frequently but reads vastly outnumber writes?
│   └── Use cache-aside with short TTL (30s-2min) + event-based invalidation
│
├── Write-heavy workload (analytics, logs)?
│   └── Use write-behind caching (buffer in Redis, flush to DB in batches)
│
└── Data must be real-time consistent?
    └── Use write-through caching or skip caching for that data
```

---

## 13. Summary

| Layer | Tool | Strategy |
|---|---|---|
| Browser | `Cache-Control` headers | Static assets + short-lived API cache |
| CDN | CloudFront / Cloudflare | Static files + edge caching |
| API Gateway | Kong proxy-cache / Nginx | Full response caching for GET endpoints |
| Application (Django) | `django.core.cache` + Redis | Cache-aside, per-view, template fragments |
| Application (FastAPI) | `fastapi-cache2` / manual Redis | Cache-aside, decorator-based, middleware |
| Database | Indexes, materialized views | Query optimization (not traditional caching) |

| Strategy | When to Use |
|---|---|
| **Cache-Aside** | Default choice — 90% of use cases |
| **Write-Through** | Strong consistency required |
| **Write-Behind** | Write-heavy, can tolerate data loss |
| **TTL + Event Invalidation** | Best invalidation approach |
| **Locking / Early Refresh** | Hot keys, high concurrency |

Caching is the **single highest-impact optimization** you can make to any system. A well-placed cache can turn a 500ms database query into a 1ms Redis lookup. But a poorly managed cache can serve stale data, waste memory, and cause outages through stampedes and avalanches. Master the strategies, understand the failure modes, and always monitor your hit rate.
