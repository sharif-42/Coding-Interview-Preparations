# CORS — Cross-Origin Resource Sharing

You built a beautiful frontend at `https://myapp.com`. It makes an API call to your backend at `https://api.myapp.com`.

You open the browser, click a button, and... nothing happens.

You open DevTools and see a big red error:

```
Access to fetch at 'https://api.myapp.com/users' from origin 'https://myapp.com'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present
on the requested resource.
```

Your server is fine. Your code is fine. The request never even fails on the server — **the browser blocks the response before your JavaScript can read it**.

Welcome to CORS.

---

## 1. What Is CORS and Why Does It Exist?

### The Same-Origin Policy

Every browser enforces a security rule called the **Same-Origin Policy (SOP)**. It says:

> A web page at one origin can only make requests to the **same** origin. Any request to a **different** origin is blocked by default.

Two URLs have the **same origin** if they share the same **protocol**, **host**, and **port**:

| URL A | URL B | Same Origin? | Why |
|---|---|---|---|
| `https://myapp.com/page` | `https://myapp.com/api` | Yes | Same protocol, host, port |
| `https://myapp.com` | `http://myapp.com` | No | Different protocol (https vs http) |
| `https://myapp.com` | `https://api.myapp.com` | No | Different host (subdomain counts) |
| `https://myapp.com` | `https://myapp.com:8080` | No | Different port (443 vs 8080) |
| `https://myapp.com` | `https://otherapp.com` | No | Completely different host |

### The Bank Analogy

Imagine you are inside a bank branch (Origin A). You want to transfer money from your account.

The bank has a strict rule: **only requests from inside this branch are trusted**. If someone standing outside on the street (Origin B) shouts through the window, "Transfer $1000 from Account 123!" — the bank ignores them.

But what if you have a legitimate partner — say, a financial advisor (Origin B) who you explicitly trust? The bank can issue a **special pass** that says: *"Requests from this advisor are allowed."*

That special pass is **CORS**.

### So What Is CORS?

**CORS (Cross-Origin Resource Sharing)** is a mechanism that allows a server to explicitly declare which other origins are permitted to access its resources.

Without CORS, the Same-Origin Policy blocks everything. With CORS, the server adds specific HTTP headers that tell the browser: *"I trust this origin. Let the response through."*

```
Without CORS:
  Browser: "Hey api.myapp.com, myapp.com wants your data."
  Server:  Returns the data.
  Browser: "No Access-Control-Allow-Origin header? BLOCKED."
           JavaScript never sees the response.

With CORS:
  Browser: "Hey api.myapp.com, myapp.com wants your data."
  Server:  Returns the data + header:
           Access-Control-Allow-Origin: https://myapp.com
  Browser: "Origin is allowed. Here's the data, JavaScript."
```

> **Critical point:** CORS is enforced by the **browser**, not the server. The server always processes the request. It is the browser that decides whether to expose the response to the frontend JavaScript. Tools like `curl`, Postman, or server-to-server calls are not affected by CORS at all.

---

![CORS Overview](/System_Design/images/cors_overview.svg)

---

## 2. How CORS Works — The Full Mechanism

There are two types of CORS requests: **Simple Requests** and **Preflight Requests**.

### 2.1 Simple Requests

A request is "simple" if it meets ALL of these criteria:

| Rule | Allowed Values |
|---|---|
| **Method** | `GET`, `HEAD`, or `POST` only |
| **Headers** | Only standard headers: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` |
| **Content-Type** | Only: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain` |

For simple requests, the browser sends the request directly and checks the `Access-Control-Allow-Origin` header in the response:

```
1. Browser sends:
   GET /api/products HTTP/1.1
   Host: api.myapp.com
   Origin: https://myapp.com       ← Browser adds this automatically

2. Server responds:
   HTTP/1.1 200 OK
   Access-Control-Allow-Origin: https://myapp.com    ← Server allows this origin
   Content-Type: application/json

   {"products": [...]}

3. Browser checks:
   Origin matches Access-Control-Allow-Origin? → Yes → Pass to JavaScript
```

### 2.2 Preflight Requests

Any request that does NOT qualify as "simple" triggers a **preflight**. This includes:

- Methods like `PUT`, `PATCH`, `DELETE`
- Custom headers like `Authorization`, `X-API-Key`
- `Content-Type: application/json` (the most common one!)

A preflight is an **extra OPTIONS request** the browser sends before the real request to ask the server: *"Is this kind of request allowed?"*

```
Step 1: Preflight (automatic — browser sends this)
──────────────────────────────────────────────────
   OPTIONS /api/orders HTTP/1.1
   Host: api.myapp.com
   Origin: https://myapp.com
   Access-Control-Request-Method: POST
   Access-Control-Request-Headers: Content-Type, Authorization

Step 2: Server responds to preflight
──────────────────────────────────────────────────
   HTTP/1.1 204 No Content
   Access-Control-Allow-Origin: https://myapp.com
   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   Access-Control-Allow-Headers: Content-Type, Authorization
   Access-Control-Max-Age: 86400      ← Cache this for 24 hours

Step 3: Browser checks preflight response
──────────────────────────────────────────────────
   Origin allowed? ✓
   Method allowed? ✓
   Headers allowed? ✓
   → Proceed with actual request

Step 4: Actual request
──────────────────────────────────────────────────
   POST /api/orders HTTP/1.1
   Host: api.myapp.com
   Origin: https://myapp.com
   Content-Type: application/json
   Authorization: Bearer <token>

   {"item": "Widget", "quantity": 2}

Step 5: Server responds normally
──────────────────────────────────────────────────
   HTTP/1.1 201 Created
   Access-Control-Allow-Origin: https://myapp.com

   {"order_id": "ORD-123"}
```

### 2.3 CORS Headers — Complete Reference

**Response headers the server sends:**

| Header | Purpose | Example |
|---|---|---|
| `Access-Control-Allow-Origin` | Which origins can access | `https://myapp.com` or `*` |
| `Access-Control-Allow-Methods` | Which HTTP methods are allowed | `GET, POST, PUT, DELETE` |
| `Access-Control-Allow-Headers` | Which request headers are allowed | `Content-Type, Authorization` |
| `Access-Control-Allow-Credentials` | Whether cookies/credentials are allowed | `true` |
| `Access-Control-Expose-Headers` | Which response headers JS can read | `X-Request-Id, X-RateLimit-Remaining` |
| `Access-Control-Max-Age` | How long to cache the preflight (seconds) | `86400` (24 hours) |

**Request headers the browser sends automatically:**

| Header | Purpose | Example |
|---|---|---|
| `Origin` | The origin of the requesting page | `https://myapp.com` |
| `Access-Control-Request-Method` | Method the actual request will use (preflight only) | `POST` |
| `Access-Control-Request-Headers` | Headers the actual request will send (preflight only) | `Content-Type, Authorization` |

### 2.4 The Credentials Problem

By default, cross-origin requests do not include cookies or authentication headers. If you need them (e.g., session-based auth), both sides must opt in:

**Server side:**
```
Access-Control-Allow-Credentials: true
```

**Client side (JavaScript):**
```javascript
fetch('https://api.myapp.com/profile', {
    credentials: 'include'    // Send cookies cross-origin
});
```

**Important restriction:** When `Allow-Credentials` is `true`, `Allow-Origin` **cannot** be `*`. You must specify the exact origin. This is a security rule — the browser rejects wildcard origins with credentials.

---

![CORS Preflight Flow](/System_Design/images/cors_preflight_flow.svg)

---

## 3. CORS in Django

Django's most popular CORS solution is the `django-cors-headers` package. It is used by virtually every Django project that serves a frontend on a separate domain.

```bash
pip install django-cors-headers
```

### 3.1 Basic Setup

```python
# settings.py

INSTALLED_APPS = [
    # ... other apps
    'corsheaders',
    # ... rest of apps
]

# IMPORTANT: Place CorsMiddleware BEFORE CommonMiddleware
# It needs to add headers before other middleware processes the response.
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',     # ← Must be high up
    'django.middleware.common.CommonMiddleware',
    'django.middleware.security.SecurityMiddleware',
    # ... rest of middleware
]
```

> **Middleware order matters.** `CorsMiddleware` must be placed as high as possible, especially before `CommonMiddleware`. If another middleware returns a response before `CorsMiddleware` runs, the CORS headers will be missing and the browser will block the request.

### 3.2 Configuration Options

#### Option 1: Allow Specific Origins (Recommended for Production)

```python
# settings.py

CORS_ALLOWED_ORIGINS = [
    "https://myapp.com",
    "https://www.myapp.com",
    "https://admin.myapp.com",
]
```

Only these three origins will receive `Access-Control-Allow-Origin` headers. Everything else is blocked.

#### Option 2: Allow All Origins (Development Only)

```python
# settings.py

CORS_ALLOW_ALL_ORIGINS = True
```

This sets `Access-Control-Allow-Origin: *` on every response. **Never use this in production** — it allows any website to call your API.

#### Option 3: Regex Pattern Matching

```python
# settings.py

CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://\w+\.myapp\.com$",          # Any subdomain of myapp.com
    r"^https://deploy-preview-\d+\.netlify\.app$",  # Netlify preview deploys
]
```

Useful when you have dynamic subdomains or preview deployments.

### 3.3 Fine-Grained Control

```python
# settings.py

# Which HTTP methods are allowed in cross-origin requests
CORS_ALLOW_METHODS = [
    "GET",
    "POST",
    "PUT",
    "PATCH",
    "DELETE",
    "OPTIONS",
]

# Which request headers the client can send
CORS_ALLOW_HEADERS = [
    "accept",
    "authorization",
    "content-type",
    "origin",
    "x-csrftoken",
    "x-requested-with",
    "x-api-key",
]

# Which response headers the client JavaScript can read
CORS_EXPOSE_HEADERS = [
    "X-Request-Id",
    "X-RateLimit-Remaining",
    "X-RateLimit-Limit",
]

# Allow cookies and credentials
CORS_ALLOW_CREDENTIALS = True

# Cache preflight responses for 1 hour (in seconds)
CORS_PREFLIGHT_MAX_AGE = 3600
```

### 3.4 Per-URL CORS Control

By default, CORS headers are added to all URLs. You can restrict CORS to specific paths:

```python
# settings.py

# Only add CORS headers to /api/ endpoints
CORS_URLS_REGEX = r"^/api/.*$"
```

This means `https://myapp.com/api/users` gets CORS headers, but `https://myapp.com/admin/` does not. Useful when your Django app serves both API endpoints and server-rendered HTML pages.

### 3.5 Complete Production Django CORS Config

```python
# settings.py — Production CORS Configuration

import os

INSTALLED_APPS = [
    'corsheaders',
    # ... other apps
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    # ... rest
]

# ── CORS Settings ──

# Explicitly list trusted origins
CORS_ALLOWED_ORIGINS = os.environ.get("CORS_ALLOWED_ORIGINS", "").split(",")
# Example env: CORS_ALLOWED_ORIGINS=https://myapp.com,https://admin.myapp.com

# Only apply CORS to API routes
CORS_URLS_REGEX = r"^/api/.*$"

# Allow credentials (cookies, Authorization header)
CORS_ALLOW_CREDENTIALS = True

# Allow standard methods
CORS_ALLOW_METHODS = [
    "GET",
    "POST",
    "PUT",
    "PATCH",
    "DELETE",
    "OPTIONS",
]

# Allow standard + custom headers
CORS_ALLOW_HEADERS = [
    "accept",
    "authorization",
    "content-type",
    "origin",
    "x-csrftoken",
    "x-requested-with",
]

# Expose rate limit headers to frontend
CORS_EXPOSE_HEADERS = [
    "X-RateLimit-Limit",
    "X-RateLimit-Remaining",
    "Retry-After",
]

# Cache preflight for 1 hour
CORS_PREFLIGHT_MAX_AGE = 3600
```

### 3.6 How django-cors-headers Works Internally

Here is what happens on every request:

```
Incoming Request
      │
      ▼
┌─────────────────────────────────┐
│      CorsMiddleware             │
│                                 │
│  1. Is it an OPTIONS request?   │
│     ├── Yes → It's a preflight  │
│     │   Check origin, method,   │
│     │   headers against config  │
│     │   Return 200 with CORS    │
│     │   headers. Stop here.     │
│     │                           │
│     └── No → Regular request    │
│         Check origin against    │
│         CORS_ALLOWED_ORIGINS    │
│         or regex patterns       │
│                                 │
│  2. Does origin match?          │
│     ├── Yes → Add headers:      │
│     │   Access-Control-Allow-   │
│     │   Origin: <origin>        │
│     │                           │
│     └── No → No CORS headers    │
│         (browser will block)    │
│                                 │
│  3. Pass to next middleware      │
└─────────────────────────────────┘
      │
      ▼
  Django View processes request
      │
      ▼
  Response returned with CORS headers
```

---

## 4. CORS in FastAPI

FastAPI handles CORS through Starlette's built-in `CORSMiddleware`. No extra packages needed — it ships with FastAPI.

### 4.1 Basic Setup

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Define allowed origins
origins = [
    "https://myapp.com",
    "https://www.myapp.com",
    "https://admin.myapp.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,           # Which origins are allowed
    allow_credentials=True,          # Allow cookies/auth headers
    allow_methods=["*"],             # Allow all HTTP methods
    allow_headers=["*"],             # Allow all headers
)
```

That is it. Four lines of middleware configuration and CORS is fully working.

### 4.2 Configuration Parameters

```python
app.add_middleware(
    CORSMiddleware,

    # ── Origins ──
    allow_origins=["https://myapp.com"],    # List of allowed origins
    # OR
    allow_origin_regex=r"https://.*\.myapp\.com",  # Regex pattern for origins

    # ── Methods ──
    allow_methods=["GET", "POST", "PUT", "DELETE"],  # Allowed HTTP methods
    # OR
    allow_methods=["*"],  # Allow all methods

    # ── Headers ──
    allow_headers=["Authorization", "Content-Type", "X-API-Key"],
    # OR
    allow_headers=["*"],  # Allow all headers

    # ── Credentials ──
    allow_credentials=True,  # Allow cookies and credentials

    # ── Expose Headers ──
    expose_headers=["X-RateLimit-Remaining", "X-Request-Id"],

    # ── Preflight Cache ──
    max_age=3600,  # Cache preflight for 1 hour
)
```

### 4.3 Common Configurations

#### Development (Allow Everything)

```python
# For local development ONLY
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,   # Cannot use credentials with wildcard origin
    allow_methods=["*"],
    allow_headers=["*"],
)
```

> **Warning:** `allow_origins=["*"]` with `allow_credentials=True` will **not work**. The browser rejects `Access-Control-Allow-Origin: *` when credentials are included. FastAPI will silently not send the `Allow-Credentials` header if origin is `*`.

#### Production (Specific Origins + Credentials)

```python
import os

app.add_middleware(
    CORSMiddleware,
    allow_origins=os.environ.get("CORS_ORIGINS", "").split(","),
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-CSRF-Token"],
    expose_headers=["X-RateLimit-Limit", "X-RateLimit-Remaining", "Retry-After"],
    max_age=3600,
)
```

#### Multiple Subdomains (Regex)

```python
app.add_middleware(
    CORSMiddleware,
    allow_origin_regex=r"https://.*\.myapp\.com",
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    max_age=7200,
)
```

### 4.4 Per-Route CORS (Advanced)

FastAPI's `CORSMiddleware` applies globally. If you need different CORS rules for different routes, you can build a custom middleware:

```python
# middleware/cors_per_route.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response
from fastapi import Request


class PerRouteCORSMiddleware(BaseHTTPMiddleware):
    """
    Apply different CORS policies based on the request path.
    """

    CORS_POLICIES = {
        "/api/public": {
            "origins": ["*"],
            "methods": ["GET"],
            "credentials": False,
        },
        "/api/private": {
            "origins": ["https://myapp.com", "https://admin.myapp.com"],
            "methods": ["GET", "POST", "PUT", "DELETE"],
            "credentials": True,
        },
        "/api/webhooks": {
            "origins": ["https://stripe.com", "https://github.com"],
            "methods": ["POST"],
            "credentials": False,
        },
    }

    async def dispatch(self, request: Request, call_next):
        origin = request.headers.get("origin", "")
        path = request.url.path

        # Find matching policy
        policy = None
        for prefix, pol in self.CORS_POLICIES.items():
            if path.startswith(prefix):
                policy = pol
                break

        # Handle preflight
        if request.method == "OPTIONS" and policy:
            if origin in policy["origins"] or "*" in policy["origins"]:
                return Response(
                    status_code=204,
                    headers={
                        "Access-Control-Allow-Origin": origin if policy["credentials"] else "*",
                        "Access-Control-Allow-Methods": ", ".join(policy["methods"]),
                        "Access-Control-Allow-Headers": "Authorization, Content-Type",
                        "Access-Control-Allow-Credentials": str(policy["credentials"]).lower(),
                        "Access-Control-Max-Age": "3600",
                    },
                )

        response = await call_next(request)

        # Add CORS headers to actual response
        if policy and (origin in policy["origins"] or "*" in policy["origins"]):
            response.headers["Access-Control-Allow-Origin"] = origin if policy["credentials"] else "*"
            if policy["credentials"]:
                response.headers["Access-Control-Allow-Credentials"] = "true"

        return response
```

Register it:

```python
from middleware.cors_per_route import PerRouteCORSMiddleware

app.add_middleware(PerRouteCORSMiddleware)
```

### 4.5 How FastAPI CORSMiddleware Works Internally

```
Incoming Request
      │
      ▼
┌──────────────────────────────────┐
│      CORSMiddleware              │
│   (Starlette built-in)          │
│                                  │
│  1. Extract Origin header        │
│                                  │
│  2. Is Origin in allow_origins   │
│     or matches allow_origin_regex?│
│     ├── No → Pass through        │
│     │   (no CORS headers added)  │
│     │                            │
│     └── Yes → Continue           │
│                                  │
│  3. Is it OPTIONS (preflight)?   │
│     ├── Yes → Return 200 with:   │
│     │   Allow-Origin             │
│     │   Allow-Methods            │
│     │   Allow-Headers            │
│     │   Max-Age                  │
│     │   (Skip the actual view)   │
│     │                            │
│     └── No → Process request     │
│         Add CORS headers to      │
│         the response:            │
│         Allow-Origin             │
│         Allow-Credentials        │
│         Expose-Headers           │
│                                  │
└──────────────────────────────────┘
      │
      ▼
  FastAPI endpoint handles request
```

---

## 5. Django vs FastAPI CORS — Comparison

| Aspect | Django (`django-cors-headers`) | FastAPI (`CORSMiddleware`) |
|---|---|---|
| **Package needed** | `django-cors-headers` (3rd party) | Built-in (Starlette) |
| **Installation** | `pip install django-cors-headers` | Nothing extra |
| **Config location** | `settings.py` (module-level variables) | `main.py` (middleware params) |
| **Allow specific origins** | `CORS_ALLOWED_ORIGINS = [...]` | `allow_origins=[...]` |
| **Allow all origins** | `CORS_ALLOW_ALL_ORIGINS = True` | `allow_origins=["*"]` |
| **Regex origins** | `CORS_ALLOWED_ORIGIN_REGEXES = [...]` | `allow_origin_regex="..."` |
| **Allow methods** | `CORS_ALLOW_METHODS = [...]` | `allow_methods=[...]` |
| **Allow headers** | `CORS_ALLOW_HEADERS = [...]` | `allow_headers=[...]` |
| **Allow credentials** | `CORS_ALLOW_CREDENTIALS = True` | `allow_credentials=True` |
| **Expose headers** | `CORS_EXPOSE_HEADERS = [...]` | `expose_headers=[...]` |
| **Preflight cache** | `CORS_PREFLIGHT_MAX_AGE = 3600` | `max_age=3600` |
| **URL filtering** | `CORS_URLS_REGEX = r"^/api/.*$"` | Manual (custom middleware) |
| **Per-route rules** | Not built-in | Not built-in (custom middleware) |
| **Middleware order** | Must be before `CommonMiddleware` | Added via `app.add_middleware()` |

The core concepts are identical. The difference is Django uses module-level settings variables while FastAPI uses middleware constructor parameters. FastAPI has the advantage of not needing an extra package.

---

## 6. Common CORS Mistakes and Fixes

### Mistake 1: Wildcard + Credentials

```
ERROR:
Access to fetch has been blocked by CORS policy: The value of the
'Access-Control-Allow-Origin' header must not be the wildcard '*'
when the request's credentials mode is 'include'.
```

**Wrong:**
```python
# Django
CORS_ALLOW_ALL_ORIGINS = True
CORS_ALLOW_CREDENTIALS = True    # ← Conflict!

# FastAPI
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],           # ← Conflict!
    allow_credentials=True,
)
```

**Fix:** Specify exact origins when using credentials.

```python
# Django
CORS_ALLOWED_ORIGINS = ["https://myapp.com"]
CORS_ALLOW_CREDENTIALS = True

# FastAPI
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
)
```

### Mistake 2: Missing Content-Type Header

```
ERROR:
Request header field content-type is not allowed by
Access-Control-Allow-Headers in preflight response.
```

This happens when you send `Content-Type: application/json` but the server does not include `Content-Type` in `Access-Control-Allow-Headers`.

**Fix:**
```python
# Django
CORS_ALLOW_HEADERS = [
    "content-type",    # ← Add this
    "authorization",
]

# FastAPI — use wildcard or list explicitly
app.add_middleware(
    CORSMiddleware,
    allow_headers=["*"],   # ← Allows any header
)
```

### Mistake 3: Middleware Order (Django)

```
ERROR: CORS headers not appearing even though django-cors-headers is installed.
```

**Wrong:**
```python
MIDDLEWARE = [
    'django.middleware.common.CommonMiddleware',
    'corsheaders.middleware.CorsMiddleware',     # ← Too late!
]
```

**Fix:** Move `CorsMiddleware` to the top:
```python
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',     # ← First!
    'django.middleware.common.CommonMiddleware',
]
```

### Mistake 4: Missing OPTIONS Method

Some servers or reverse proxies strip the `OPTIONS` method. The preflight never reaches your Django/FastAPI app.

**Fix:** Ensure your Nginx/reverse proxy passes OPTIONS through:
```nginx
location /api/ {
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    proxy_pass http://backend;
}
```

Or better — let the backend handle OPTIONS and remove Nginx CORS entirely.

### Mistake 5: Forgetting `expose_headers`

Your API returns a custom header like `X-Request-Id`, but your frontend JavaScript cannot read it:

```javascript
const response = await fetch('https://api.myapp.com/data');
console.log(response.headers.get('X-Request-Id'));  // null!
```

By default, JavaScript can only read these "safe" response headers: `Cache-Control`, `Content-Language`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`.

**Fix:**
```python
# Django
CORS_EXPOSE_HEADERS = ["X-Request-Id", "X-RateLimit-Remaining"]

# FastAPI
app.add_middleware(
    CORSMiddleware,
    expose_headers=["X-Request-Id", "X-RateLimit-Remaining"],
)
```

---

## 7. CORS at the API Gateway Level

If you are running multiple microservices behind an API Gateway (Kong, Nginx, Traefik), you have a choice:

| Approach | Configure CORS in... | Pros | Cons |
|---|---|---|---|
| **Gateway-level** | API Gateway only | Centralized, consistent, one config | Less granular per-service control |
| **Service-level** | Each microservice | Fine-grained per-service rules | Duplicated config, harder to maintain |
| **Both** | Gateway + Services | Defense in depth | Complex, risk of conflicting headers |

### Recommended: Gateway-Level CORS

Just like rate limiting, CORS is a **cross-cutting concern** that belongs at the edge. The gateway handles it once, and individual services do not need to worry about it.

#### Kong CORS Plugin

```yaml
# kong.yml
plugins:
  - name: cors
    config:
      origins:
        - "https://myapp.com"
        - "https://admin.myapp.com"
      methods:
        - GET
        - POST
        - PUT
        - PATCH
        - DELETE
      headers:
        - Authorization
        - Content-Type
        - X-API-Key
      exposed_headers:
        - X-RateLimit-Limit
        - X-RateLimit-Remaining
      credentials: true
      max_age: 3600
      preflight_continue: false   # Gateway handles preflight, don't pass to upstream
```

#### Nginx CORS Headers

```nginx
server {
    listen 80;

    # Handle preflight globally
    location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' $http_origin;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Max-Age' 3600;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' $http_origin always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;

        proxy_pass http://backend;
    }
}
```

> **If using gateway-level CORS:** Remove CORS middleware from your Django/FastAPI apps to avoid duplicate `Access-Control-Allow-Origin` headers. Some browsers reject responses with duplicate CORS headers.

---

## 8. CORS Debugging Checklist

When you hit a CORS error, walk through this checklist:

- [ ] **Check the exact error message** in the browser DevTools Console — it tells you exactly what is missing
- [ ] **Verify the origin** — is the requesting origin listed in your allowed origins?
- [ ] **Check protocol** — `http://` and `https://` are different origins
- [ ] **Check port** — `localhost:3000` and `localhost:8000` are different origins
- [ ] **Is it a preflight issue?** — Check the Network tab for an `OPTIONS` request. Is it returning the right headers?
- [ ] **Middleware order** (Django) — Is `CorsMiddleware` at the top of the middleware list?
- [ ] **Credentials + wildcard?** — You cannot use `allow_origins=["*"]` with `allow_credentials=True`
- [ ] **Custom headers blocked?** — Does `Access-Control-Allow-Headers` include all headers your client sends?
- [ ] **Response headers missing?** — Add them to `expose_headers`
- [ ] **Reverse proxy stripping headers?** — Check Nginx/Apache/CDN is not removing CORS headers
- [ ] **Duplicate headers?** — Both gateway and service adding CORS headers = double values = error

---

## 9. Summary

| Concept | What It Is |
|---|---|
| **Same-Origin Policy** | Browser security rule: block cross-origin requests by default |
| **CORS** | Server-side headers that whitelist specific origins to bypass SOP |
| **Simple Request** | GET/HEAD/POST with standard headers — no preflight needed |
| **Preflight** | OPTIONS request before non-simple requests — browser asks permission |
| **`Access-Control-Allow-Origin`** | The most important header — specifies which origin is allowed |
| **Credentials** | Cookies/auth headers require `Allow-Credentials: true` + specific origin (no wildcard) |

| Scope | Tool | Key Config |
|---|---|---|
| Django | `django-cors-headers` | `CORS_ALLOWED_ORIGINS` in settings.py |
| FastAPI | Built-in `CORSMiddleware` | `allow_origins` in `app.add_middleware()` |
| API Gateway | Kong CORS plugin / Nginx headers | Centralized — remove CORS from services |

CORS is not optional. Any modern web application with a separate frontend and backend will hit CORS immediately. Understand the mechanism, configure it correctly from day one, and you will never waste hours debugging that red error in DevTools again.
