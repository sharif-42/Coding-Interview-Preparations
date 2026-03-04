# How to Architect an API Gateway

Every modern application you use — whether it is Uber, Netflix, or Shopify — has dozens or even hundreds of microservices running behind the scenes.

Your mobile app does not talk to each one individually. That would be chaos.

Instead, there is a single door through which every request enters. That door is called an **API Gateway**.

It is one of the most critical components in any microservices architecture. It handles routing, authentication, rate limiting, load balancing, and much more — all in one place.

In this document, we will break down what an API Gateway is, why it exists, the different types and patterns, and how to architect one for a real production system.

---

## 1. What Is an API Gateway?

An **API Gateway** is a server that acts as the **single entry point** for all client requests into a microservices-based system.

`Instead of clients calling each microservice directly, they call the API Gateway, and the gateway figures out where to route each request.`

Think of it as a **smart receptionist** for your entire backend.

### The Hotel Concierge Analogy

You check into a luxury hotel. You need:
- A restaurant reservation
- A taxi to the airport
- Your laundry picked up
- A wake-up call at 6 AM

You do not call the restaurant, the taxi company, the laundry room, and the front desk separately.

You call the **concierge**. One phone call. The concierge handles everything behind the scenes — talks to the right departments, coordinates timing, and calls you back with a single confirmation.

The concierge is the **API Gateway**. The client makes one call, and the gateway routes it to the appropriate backend services.

### Why Not Call Microservices Directly?

Without an API gateway, a mobile app loading a food delivery homepage might need to call:

| Service | Endpoint | Purpose |
|---|---|---|
| User Service | `user-svc:8001/api/profile` | Get user profile |
| Restaurant Service | `restaurant-svc:8002/api/nearby` | Get nearby restaurants |
| Promo Service | `promo-svc:8003/api/offers` | Get active promotions |
| Order Service | `order-svc:8004/api/recent` | Get recent orders |
| Cart Service | `cart-svc:8005/api/items` | Get current cart |

That is **5 different hosts, 5 different ports, 5 different calls** from the client.

This creates serious problems:

- **Tight coupling** — The client must know the address of every microservice.
- **Security nightmare** — Every service is exposed to the public internet.
- **No centralized auth** — Each service must independently validate tokens.
- **Cross-cutting concerns duplicated** — Logging, rate limiting, CORS — implemented in every service.
- **Mobile performance** — 5 round trips over a cellular network is painfully slow.

An API Gateway solves all of this.

---

![API Gateway Overview](/System_Design/images/api_gateway_overview.svg)

---

## 2. Core Responsibilities of an API Gateway

An API Gateway is not just a router. It is a **multi-functional layer** that handles many cross-cutting concerns:

| Responsibility | What It Does |
|---|---|
| **Request Routing** | Routes `/api/users` to the User Service, `/api/orders` to the Order Service, etc. |
| **Authentication & Authorization** | Validates JWT tokens, API keys, or OAuth tokens before the request ever reaches a backend service. |
| **Rate Limiting & Throttling** | Enforces limits like "100 requests per minute per user" to prevent abuse. |
| **Load Balancing** | Distributes traffic across multiple instances of the same service. |
| **Request/Response Transformation** | Converts between protocols (e.g., REST to gRPC), adds/removes headers, reshapes payloads. |
| **API Composition / Aggregation** | Combines responses from multiple services into a single response for the client. |
| **Caching** | Caches frequently requested data at the gateway level to reduce backend load. |
| **Circuit Breaking** | Stops forwarding requests to a failing service to prevent cascading failures. |
| **SSL/TLS Termination** | Handles HTTPS encryption so backend services can communicate over plain HTTP internally. |
| **Logging & Monitoring** | Centralized request logging, metrics collection, and distributed tracing. |
| **IP Whitelisting / Blacklisting** | Allows or blocks traffic based on client IP addresses. |
| **Request Validation** | Validates request schemas before forwarding — reject malformed requests early. |
| **CORS Handling** | Manages Cross-Origin Resource Sharing policies in one place. |
| **Versioning** | Routes `/v1/users` to the old service and `/v2/users` to the new one. |

---

## 3. API Gateway Architecture Patterns

There are several architectural patterns for how API Gateways are designed and deployed. Each pattern suits different scales and requirements.

![API Gateway Patterns](/System_Design/images/api_gateway_patterns.svg)

### Pattern 1: Single Gateway (Monolithic Gateway)

The simplest pattern. One API Gateway handles **all** traffic for **all** clients and **all** services.

```
Mobile App  ─┐
Web App     ─┤──►  [ Single API Gateway ]  ──► Microservices
Third-party ─┘
```

**When to use:**
- Small to medium-sized systems (< 20 microservices).
- A single team owns the gateway.
- Client types have similar needs.

**Pros:**
- Simple to deploy and manage.
- One place for all cross-cutting concerns.

**Cons:**
- Becomes a bottleneck as the system grows.
- One team becomes a blocker for all service teams.
- A single deployment failure can take down everything.

---

### Pattern 2: Backend for Frontend (BFF)

Each client type gets its **own dedicated API Gateway**, tailored to its specific needs.

```
Mobile App  ──►  [ Mobile BFF Gateway ]  ──►  Microservices
Web App     ──►  [ Web BFF Gateway ]     ──►  Microservices
Third-party ──►  [ Partner API Gateway ] ──►  Microservices
```

**Why?** A mobile app and a web app have very different needs:

| Concern | Mobile App | Web App |
|---|---|---|
| **Payload size** | Minimal (save bandwidth) | Larger is OK |
| **Aggregation** | Combine into 1 call (fewer round trips) | Can make parallel calls |
| **Auth** | Bearer token | Session cookie or JWT |
| **Response format** | Compact JSON | Full JSON with nested objects |
| **API versioning** | Must support old app versions | Always latest |

**When to use:**
- Multiple distinct client types (mobile, web, IoT, partner APIs).
- Each client needs different response shapes or aggregation logic.
- Different teams own different client experiences.

**Pros:**
- Each gateway is optimized for its client.
- Teams can deploy independently.
- A mobile gateway failure does not affect the web app.

**Cons:**
- More infrastructure to manage.
- Some logic is duplicated across gateways.

Netflix, SoundCloud, and Spotify use this pattern.

---

### Pattern 3: Gateway with Sidecar (Service Mesh)

Instead of a centralized gateway, each microservice gets its own **sidecar proxy** (e.g., Envoy). A lightweight **edge gateway** handles external traffic, while the sidecars handle service-to-service communication.

```
Client  ──►  [ Edge Gateway ]  ──►  [ Sidecar ↔ Service A ]
                                     [ Sidecar ↔ Service B ]
                                     [ Sidecar ↔ Service C ]
```

**How it works:**
- The **edge gateway** handles external concerns: SSL, auth, rate limiting.
- The **sidecar proxy** handles internal concerns: service discovery, retries, circuit breaking, mTLS, observability.
- Tools like **Istio** (control plane) + **Envoy** (data plane) implement this pattern.

**When to use:**
- Large-scale systems (100+ microservices).
- You need fine-grained control over inter-service communication.
- Security and observability are critical (e.g., financial systems, healthcare).

**Pros:**
- Decentralized — no single bottleneck.
- Each service gets its own traffic management.
- Language-agnostic (sidecar handles networking, not your app code).

**Cons:**
- Complex to set up and operate.
- Adds latency (extra network hop per sidecar).
- Requires a dedicated platform team.

Google, Lyft, and Airbnb use service mesh architectures.

---

### Pattern 4: Gateway as a Federated GraphQL Layer

The gateway exposes a single **GraphQL endpoint** that federates queries across multiple microservices.

```
Client  ──►  [ GraphQL Gateway (Apollo Federation) ]
                    │           │           │
              User Service  Order Service  Product Service
              (subgraph)    (subgraph)     (subgraph)
```

**How it works:**
- Each microservice exposes a **GraphQL subgraph** defining its part of the schema.
- The gateway composes them into a single **supergraph**.
- The client sends one GraphQL query, and the gateway fetches data from multiple services automatically.

**Example query:**
```graphql
query {
  user(id: "123") {
    name
    email
    orders {
      id
      total
      items {
        productName
        price
      }
    }
  }
}
```

This single query hits User Service, Order Service, and Product Service — but the client makes **one request**.

**When to use:**
- Client-heavy applications where frontend teams need flexibility.
- Complex data relationships spanning multiple services.
- You want to avoid over-fetching and under-fetching.

**Pros:**
- Clients get exactly the data they need — nothing more, nothing less.
- Single request, multiple services.
- Strong typing and introspection.

**Cons:**
- Adds complexity to each service (must expose a GraphQL subgraph).
- Query optimization is harder (N+1 problems).
- Caching is more complex than REST.

GitHub, Shopify, and Netflix use federated GraphQL.

---

## 4. Popular API Gateway Solutions Compared

| Gateway | Type | Protocol | Best For | Managed? |
|---|---|---|---|---|
| **Kong** | Open-source / Enterprise | REST, gRPC, GraphQL | General purpose, plugin ecosystem | Self-hosted or Kong Cloud |
| **AWS API Gateway** | Cloud-managed | REST, WebSocket, HTTP | AWS-native apps, serverless (Lambda) | Fully managed |
| **Nginx** | Open-source | HTTP, TCP, gRPC | High performance, custom configs | Self-hosted |
| **Traefik** | Open-source | HTTP, TCP, gRPC | Docker/Kubernetes-native auto-discovery | Self-hosted |
| **Envoy** | Open-source | HTTP/2, gRPC, TCP | Service mesh (Istio sidecar) | Self-hosted |
| **Apollo Gateway** | Open-source | GraphQL | Federated GraphQL | Self-hosted or Apollo Cloud |
| **Azure API Management** | Cloud-managed | REST, SOAP, GraphQL | Azure-native apps, enterprise | Fully managed |
| **Apigee (Google)** | Cloud-managed | REST, gRPC | Enterprise API management, analytics | Fully managed |
| **KrakenD** | Open-source | REST, GraphQL | Ultra-high performance, stateless | Self-hosted |
| **Tyk** | Open-source / Enterprise | REST, GraphQL, gRPC | Full lifecycle API management | Self-hosted or Cloud |

### How to Choose?

| If you need... | Consider... |
|---|---|
| Simplest setup, AWS native | **AWS API Gateway** |
| Maximum plugin ecosystem | **Kong** |
| Kubernetes-native auto-discovery | **Traefik** |
| Raw performance, custom routing | **Nginx** or **KrakenD** |
| Service mesh (inter-service) | **Envoy** + Istio |
| GraphQL federation | **Apollo Gateway** |
| Enterprise API management | **Apigee** or **Azure API Management** |

---

## 5. Practical Example: Architecting an API Gateway for a Food Delivery App

Let's design a complete API Gateway architecture for **FoodDash**, a food delivery platform with 6 microservices.

![FoodDash API Gateway Request Flow](/System_Design/images/api_gateway_request_flow.svg)

### The Architecture

**Client types:**
- Mobile App (iOS/Android)
- Web App (React SPA)
- Partner Dashboard (Restaurant owners)

**Microservices:**

| Service | Internal Address | Port | Purpose |
|---|---|---|---|
| User Service | `user-svc.internal` | 8001 | Auth, profiles, addresses |
| Restaurant Service | `restaurant-svc.internal` | 8002 | Menus, restaurant info, search |
| Order Service | `order-svc.internal` | 8003 | Place, track, cancel orders |
| Payment Service | `payment-svc.internal` | 8004 | Process payments, refunds |
| Notification Service | `notification-svc.internal` | 8005 | Push notifications, SMS, email |
| Delivery Service | `delivery-svc.internal` | 8006 | Driver assignment, live tracking |

**API Gateway:** Kong Gateway running on `api.fooddash.com` (IP: `203.0.113.200`)

### Kong Gateway Configuration

```yaml
# kong.yml - Declarative Configuration

_format_version: "3.0"

# ===== SERVICES =====
services:
  - name: user-service
    url: http://user-svc.internal:8001
    routes:
      - name: user-routes
        paths:
          - /api/v1/users
          - /api/v1/auth
        strip_path: false

  - name: restaurant-service
    url: http://restaurant-svc.internal:8002
    routes:
      - name: restaurant-routes
        paths:
          - /api/v1/restaurants
          - /api/v1/menus
        strip_path: false

  - name: order-service
    url: http://order-svc.internal:8003
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders
        strip_path: false

  - name: payment-service
    url: http://payment-svc.internal:8004
    routes:
      - name: payment-routes
        paths:
          - /api/v1/payments
        strip_path: false

  - name: notification-service
    url: http://notification-svc.internal:8005
    routes:
      - name: notification-routes
        paths:
          - /api/v1/notifications
        strip_path: false

  - name: delivery-service
    url: http://delivery-svc.internal:8006
    routes:
      - name: delivery-routes
        paths:
          - /api/v1/delivery
          - /api/v1/tracking
        strip_path: false

# ===== PLUGINS (Cross-cutting concerns) =====
plugins:
  # JWT Authentication on all routes
  - name: jwt
    config:
      uri_param_names:
        - jwt
      claims_to_verify:
        - exp

  # Rate limiting: 100 requests/min per consumer
  - name: rate-limiting
    config:
      minute: 100
      policy: redis
      redis_host: redis.internal

  # Request logging
  - name: file-log
    config:
      path: /var/log/kong/requests.log

  # CORS for web clients
  - name: cors
    config:
      origins:
        - "https://app.fooddash.com"
        - "https://partners.fooddash.com"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
      headers:
        - Authorization
        - Content-Type
      max_age: 3600

# ===== CONSUMERS (API clients) =====
consumers:
  - username: mobile-app
    jwt_secrets:
      - key: mobile-app-key
        algorithm: HS256

  - username: web-app
    jwt_secrets:
      - key: web-app-key
        algorithm: HS256

  - username: partner-dashboard
    jwt_secrets:
      - key: partner-key
        algorithm: HS256
```

### Step-by-Step: What Happens When a User Opens the App

**Scenario:** A user opens the FoodDash mobile app and views nearby restaurants.

**Step 1 — App makes a request**

```
GET https://api.fooddash.com/api/v1/restaurants/nearby?lat=23.8&lng=90.4
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
```

**Step 2 — DNS resolves to the API Gateway**

`api.fooddash.com` → `203.0.113.200` (Kong Gateway)

**Step 3 — Kong processes the request through its plugin chain**

| Order | Plugin | Action |
|---|---|---|
| 1 | **SSL Termination** | Decrypts HTTPS → internal HTTP |
| 2 | **IP Restriction** | Client IP is not blacklisted → proceed |
| 3 | **Rate Limiting** | User has made 12 requests this minute (limit: 100) → proceed |
| 4 | **JWT Auth** | Validates token, extracts `user_id: 42` → proceed |
| 5 | **Request Transformer** | Adds `X-User-ID: 42` header for the backend |
| 6 | **Route Matching** | Path `/api/v1/restaurants/nearby` matches restaurant-service |
| 7 | **Load Balancing** | 3 instances of restaurant-service → picks one (round robin) |
| 8 | **Logging** | Logs: timestamp, method, path, user_id, response_time |

**Step 4 — Request reaches the Restaurant Service**

```
GET http://restaurant-svc.internal:8002/api/v1/restaurants/nearby?lat=23.8&lng=90.4
X-User-ID: 42
X-Request-ID: abc-123-def
```

The service does not need to validate the JWT — the gateway already did that. It simply reads `X-User-ID` and processes the query.

**Step 5 — Response flows back through the gateway**

```json
{
  "restaurants": [
    {"id": 1, "name": "Pizza Palace", "distance": "0.5km", "rating": 4.8},
    {"id": 2, "name": "Burger Barn", "distance": "0.8km", "rating": 4.5},
    {"id": 3, "name": "Sushi Spot", "distance": "1.2km", "rating": 4.9}
  ]
}
```

Kong adds response headers (`X-RateLimit-Remaining: 88`), compresses the response (gzip), and sends it back to the mobile app over HTTPS.

### Tracing a Complete Order Flow

When a user places an order, the gateway routes to **multiple services** sequentially:

| Step | Client Request | Gateway Routes To | What Happens |
|---|---|---|---|
| 1 | `POST /api/v1/orders` | Order Service | Creates order, returns `order_id: 789` |
| 2 | `POST /api/v1/payments` | Payment Service | Charges the user's card |
| 3 | (Internal) | Notification Service | Order Service triggers push notification via internal event |
| 4 | `GET /api/v1/tracking/789` | Delivery Service | Client polls for live driver location |

Each call goes through the **same gateway**, getting the same auth validation, rate limiting, and logging — without any service needing to implement these independently.

---

## 6. API Gateway vs Reverse Proxy vs Load Balancer

These three are often confused because they overlap. Here is how they differ:

![API Gateway vs Reverse Proxy vs Load Balancer](/System_Design/images/api_gateway_comparison.svg)

| Feature | API Gateway | Reverse Proxy | Load Balancer |
|---|---|---|---|
| **Primary purpose** | API management, routing, auth, composition | Hide backend servers, SSL, caching | Distribute traffic evenly |
| **Protocol awareness** | HTTP, gRPC, WebSocket, GraphQL | HTTP, TCP | TCP/HTTP |
| **Authentication** | Yes (JWT, OAuth, API keys) | Rarely | No |
| **Rate limiting** | Yes (per user, per route, per API key) | Basic (per IP) | No |
| **Request transformation** | Yes (modify headers, body, protocol) | Basic (headers) | No |
| **API composition** | Yes (aggregate multiple services) | No | No |
| **Service discovery** | Yes (auto-discover services) | Manual configuration | Configured pool |
| **Circuit breaking** | Yes | No | Basic (health checks) |
| **Caching** | Yes (per-route, per-user) | Yes (per-URL) | No |
| **Analytics & monitoring** | Yes (per-API, per-consumer metrics) | Basic (access logs) | Basic |
| **API versioning** | Yes (route /v1 vs /v2) | Manual | No |
| **Typical tools** | Kong, AWS API Gateway, Apigee | Nginx, Traefik, Caddy | HAProxy, AWS NLB/ALB |

> **Key Insight:** A reverse proxy forwards requests. A load balancer distributes requests. An API Gateway **understands** requests — it authenticates, authorizes, transforms, rate-limits, aggregates, and observes them. An API Gateway is essentially a smart, application-aware reverse proxy with built-in API management.

---

## 7. Anti-Patterns to Avoid

When designing an API Gateway, these are common mistakes that can hurt your system:

### ❌ 1. Putting Business Logic in the Gateway

The gateway should only handle **cross-cutting concerns** (auth, routing, rate limiting). If you start writing business rules like "apply a 20% discount if the user is premium," you are turning the gateway into a monolith.

**Rule:** The gateway routes requests. Services make decisions.

### ❌ 2. Single Point of Failure

If your gateway goes down, **everything** goes down. Always deploy the gateway as a **cluster** with multiple instances behind a load balancer.

```
Client  ──►  [ LB ]  ──►  Gateway Instance 1
                     ──►  Gateway Instance 2
                     ──►  Gateway Instance 3
```

### ❌ 3. Gateway Becomes a Bottleneck

If every request must pass through a single gateway, and the gateway does too much processing (heavy transformations, complex aggregation), it becomes the system's bottleneck.

**Solution:** Use the BFF pattern — split into multiple gateways per client type. Keep each one lightweight.

### ❌ 4. No Circuit Breaking

If a backend service is down, the gateway keeps forwarding requests to it, causing timeouts and cascading failures.

**Solution:** Configure circuit breakers. After N consecutive failures, the gateway should **stop** forwarding to that service and return a fallback response (e.g., cached data, graceful error).

### ❌ 5. Skipping Rate Limiting

Without rate limiting, a single misbehaving client can overwhelm your entire backend.

**Solution:** Always configure rate limits — per user, per API key, per route. Use tiered limits for different consumer types.

---

## 8. Production Checklist

Before deploying an API Gateway to production, make sure you have:

| Category | Checklist Item |
|---|---|
| **High Availability** | ☐ Deploy multiple gateway instances behind a load balancer |
| **High Availability** | ☐ Configure health checks for the gateway itself |
| **Security** | ☐ Enable SSL/TLS termination |
| **Security** | ☐ Configure JWT / OAuth / API key authentication |
| **Security** | ☐ Set up IP whitelisting / blacklisting |
| **Security** | ☐ Enable CORS with specific allowed origins |
| **Performance** | ☐ Configure rate limiting per consumer |
| **Performance** | ☐ Enable response caching for read-heavy endpoints |
| **Performance** | ☐ Enable gzip/brotli compression |
| **Resilience** | ☐ Configure circuit breakers for each backend service |
| **Resilience** | ☐ Set request timeouts (don't wait forever for slow services) |
| **Resilience** | ☐ Configure retry policies with exponential backoff |
| **Observability** | ☐ Enable centralized logging (ELK, Datadog, etc.) |
| **Observability** | ☐ Enable distributed tracing (Jaeger, Zipkin) |
| **Observability** | ☐ Set up dashboards for latency, error rate, throughput |
| **Operations** | ☐ Use declarative config (version-controlled YAML) |
| **Operations** | ☐ Automate deployment via CI/CD |
| **Operations** | ☐ Test with load testing before launch (k6, Locust) |

---

## 9. Quick Decision Guide

**When do I need an API Gateway?**
- You have **multiple microservices** and want a single entry point.
- You need centralized **authentication, rate limiting, or logging**.
- You have **multiple client types** (mobile, web, partner APIs).

**When do I NOT need an API Gateway?**
- You have a **monolithic application** (one service handles everything).
- You only have **1-2 services** with a simple load balancer in front.
- Your internal services use a **service mesh** for all communication (gateway only needed at the edge).

**Which pattern should I choose?**

| Your Situation | Recommended Pattern |
|---|---|
| Small team, < 20 services | Single Gateway (Kong, Nginx) |
| Multiple client types with different needs | BFF Pattern (one gateway per client) |
| 100+ services, complex inter-service communication | Service Mesh (Envoy + Istio) + Edge Gateway |
| Frontend-driven, flexible data needs | Federated GraphQL (Apollo Gateway) |

---

## Summary

| Concept | One-Liner |
|---|---|
| **API Gateway** | Single entry point to a microservices system that handles routing, auth, rate limiting, and more. |
| **Single Gateway** | One gateway for everything — simple but can become a bottleneck. |
| **BFF Pattern** | Dedicated gateway per client type — optimized for each client's needs. |
| **Service Mesh** | Sidecar proxies per service + lightweight edge gateway — decentralized, highly scalable. |
| **GraphQL Federation** | Gateway composes a single GraphQL schema from multiple service subgraphs. |
| **Key Rule** | The gateway should handle cross-cutting concerns only. Never put business logic in the gateway. |