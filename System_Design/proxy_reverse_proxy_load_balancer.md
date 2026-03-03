# Proxy, Forward Proxy, Reverse Proxy & Load Balancer

Every time you open a website, your request does not always go straight to the server.

There are often invisible middlemen sitting between you and the destination. These middlemen intercept, route, filter, or distribute your requests.

They are called **proxies** and **load balancers**.

Understanding how they work is one of the most important things in system design. Whether you are designing a CDN, scaling a web application, or securing an internal network, these concepts show up again and again.

Let's break each one down, see how they work under the hood, and then compare them side by side.

---

## 1. Proxy (The General Concept)

A **proxy** is simply an intermediary that sits between two parties in a network communication and acts on behalf of one of them.

Think of it like a middleman.

`Instead of Party A talking directly to Party B, Party A talks to the proxy, and the proxy talks to Party B.`

The proxy can inspect, modify, cache, or block the communication.

### The Receptionist Analogy

Imagine you walk into a large corporate office to meet someone.

You do not just walk through the building and knock on doors.

Instead, you go to the **receptionist** at the front desk.

**You:** "I need to speak with Bob from engineering."

**Receptionist:** "Let me check if Bob is available."

The receptionist calls Bob, confirms he is free, and then directs you.

In this scenario, the receptionist is the **proxy**. She is the intermediary between you (the client) and Bob (the server). You never interact directly.

### Why Use a Proxy?

- **Security:** Hide the identity of the client or server.
- **Filtering:** Block access to certain resources.
- **Caching:** Store and serve frequently accessed content.
- **Logging & Monitoring:** Track all requests flowing through.
- **Performance:** Compress or optimize data in transit.

There are two fundamental flavors of proxy: **Forward Proxy** and **Reverse Proxy**.

---

## 2. Forward Proxy

A **forward proxy** sits in front of the **clients** and acts on their behalf. The client knows about the proxy and is configured to send requests through it.

The server on the other side usually has **no idea** that a proxy is involved. It just sees a request coming from the proxy's IP address.

![Forward Proxy](/System_Design/images/forward_proxy.svg)

### The Personal Assistant Analogy

Imagine you are a busy CEO. You do not make phone calls yourself. You tell your **personal assistant**:

**You:** "Call the restaurant and book a table for four at 8 PM."

**Assistant:** Makes the call.

**Restaurant:** "Sure, table booked for your assistant's phone number."

The restaurant never hears your voice. They never get your phone number. They only interact with your assistant.

Your assistant is the **forward proxy**. She acts **on your behalf**, and the destination (the restaurant) does not know who the real requester is.

### How It Works

1. The client (e.g., your browser or an application) is **explicitly configured** to use the forward proxy.
2. When the client wants to reach `example.com`, it sends the request to the proxy instead.
3. The proxy evaluates the request — it may allow it, block it, cache it, or modify it.
4. If allowed, the proxy forwards the request to `example.com` on behalf of the client.
5. The response from `example.com` comes back to the proxy.
6. The proxy forwards the response back to the client.

### Real-World Use Cases

| Use Case | Description |
|---|---|
| **Corporate Networks** | Companies use forward proxies to restrict employees from accessing certain websites (e.g., social media during work hours). |
| **Privacy / Anonymity** | Users route traffic through a proxy to hide their real IP address from destination servers (e.g., VPNs act as forward proxies at the network level). |
| **Content Filtering** | Schools and libraries use forward proxies to block inappropriate content. |
| **Caching** | A proxy can cache frequently accessed pages so that repeated requests are served locally without hitting the origin server. |
| **Bypassing Geo-Restrictions** | Users in one country route traffic through a proxy in another country to access geo-blocked content. |

### Practical Example: Forward Proxy with 4 Destination Servers

Let's say you work at a company called **Acme Corp**. The company has 4 employees on a private network (`192.168.1.x`), and they need to access external services on the internet. A **Squid forward proxy** sits at the edge of the corporate network.

![Forward Proxy with 4 Destination Servers](/System_Design/images/forward_proxy_4_servers.svg)

**Your infrastructure:**

| Employee | Private IP | Role |
|---|---|---|
| Alice | `192.168.1.10` | Developer |
| Bob | `192.168.1.11` | Designer |
| Carol | `192.168.1.12` | Manager |
| Dave | `192.168.1.13` | Intern |

**The Forward Proxy (Squid):** Internal IP `192.168.1.1`, External IP `198.51.100.5`

**The 4 external servers they want to reach:**

| Server | Public IP | Service | Proxy Policy |
|---|---|---|---|
| Server 1 | `140.82.121.4` | GitHub API | ✅ Allowed |
| Server 2 | `151.101.1.69` | Stack Overflow | ✅ Allowed |
| Server 3 | `52.217.44.0` | AWS S3 | ✅ Allowed |
| Server 4 | `157.240.1.35` | Facebook | 🚫 Blocked |

Every employee's browser is configured to use the proxy at `192.168.1.1:3128`.

#### Step-by-Step: What Happens When Alice Visits GitHub

**Step 1 — Browser sends request to the proxy**

Alice types `github.com` in her browser. Instead of going directly to GitHub, the browser sends the request to the Squid proxy at `192.168.1.1:3128` because it is configured as the HTTP proxy.

**Step 2 — Proxy evaluates the request**

Squid checks its access rules:

```squid
# squid.conf
acl blocked_sites dstdomain .facebook.com .tiktok.com .youtube.com
acl localnet src 192.168.1.0/24

http_access deny blocked_sites
http_access allow localnet
http_access deny all

# Cache popular responses for 1 hour
refresh_pattern . 0 20% 60
cache_dir ufs /var/spool/squid 10000 16 256
```

`github.com` is not in the `blocked_sites` list → request is **allowed**.

**Step 3 — Proxy forwards the request**

Squid makes the request to GitHub (`140.82.121.4`) **on behalf of Alice**, but using the proxy's external IP `198.51.100.5`.

**Step 4 — GitHub responds**

GitHub sees a request from `198.51.100.5`. It has no idea Alice (`192.168.1.10`) exists. It sends the response back to the proxy.

**Step 5 — Proxy returns the response to Alice**

Squid receives the response, optionally caches it, and forwards it back to Alice's browser.

#### Tracing 4 Different Requests

| # | Employee | Request | Proxy Decision | What Server Sees |
|---|---|---|---|---|
| 1 | Alice | `GET github.com/api/repos` | ✅ Allowed → forward to `140.82.121.4` | Request from `198.51.100.5` |
| 2 | Bob | `GET stackoverflow.com/questions` | ✅ Allowed → forward to `151.101.1.69` | Request from `198.51.100.5` |
| 3 | Carol | `GET s3.amazonaws.com/backup.zip` | ✅ Allowed → forward to `52.217.44.0` | Request from `198.51.100.5` |
| 4 | Dave | `GET facebook.com/feed` | 🚫 **BLOCKED** → returns 403 to Dave | Facebook never receives the request |

#### What the Server Sees vs What Actually Happens

| | Server's View | Reality |
|---|---|---|
| **Source IP** | `198.51.100.5` (the proxy) | 4 different employees with different IPs |
| **Number of clients** | Looks like 1 client | 4 different people |
| **Client identity** | Hidden | Alice, Bob, Carol, Dave |
| **Blocked requests** | Never arrives at server | Proxy returns 403 to the employee |
| **Cached responses** | Fewer requests hit the server | Squid serves cached responses locally |

This is the power of a forward proxy: it gives the organization **full control** over outbound traffic, hides employee identities from external servers, caches common resources, and enforces access policies — all transparently.

### Key Characteristics

- The **client knows** about the proxy and is configured to use it.
- The **server does not know** about the proxy. It sees requests from the proxy's IP.
- It protects and anonymizes the **client side**.

---

## 3. Reverse Proxy

A **reverse proxy** sits in front of the **servers** and acts on their behalf. The client usually has **no idea** a reverse proxy is involved. It thinks it is talking directly to the actual server.

![Reverse Proxy](/System_Design/images/reverse_proxy.svg)

### The Restaurant Host Analogy

You walk into a restaurant.

You do not go to the kitchen and tell the chef what you want.

Instead, you talk to the **host** at the entrance.

**You:** "Table for two, please."

**Host:** Checks which tables are available, picks the best one, and seats you.

You never see the behind-the-scenes logistics. You do not know that Table 5 was just cleaned and Table 8 has a wobbly leg. The host handles all of that.

`The host is the reverse proxy.` It sits in front of the servers (tables/kitchen), receives all incoming requests (guests), and intelligently routes them.

### How It Works

1. The client sends a request to what it believes is the actual server (e.g., `api.example.com`).
2. The DNS for `api.example.com` actually resolves to the **reverse proxy's IP**, not the origin server.
3. The reverse proxy receives the request.
4. It decides which backend server should handle the request (based on rules, health checks, load, etc.).
5. It forwards the request to the chosen backend server.
6. The backend server processes the request and sends the response to the reverse proxy.
7. The reverse proxy forwards the response to the client.

The client never knows the real IP or identity of the backend servers.

### Practical Example: Reverse Proxy with 4 Backend Servers

Let's say you are running an e-commerce platform called **shop.example.com**. You have 4 backend servers, each with a different private IP and a different responsibility.

![Reverse Proxy with 4 Backend Servers](/System_Design/images/reverse_proxy_4_servers.svg)

**Your infrastructure:**

| Server | Private IP | Role | What It Runs |
|---|---|---|---|
| Server 1 | `10.0.1.10` | API (Primary) | Django REST Framework on port 8000 |
| Server 2 | `10.0.1.11` | API (Replica) | Django REST Framework on port 8000 |
| Server 3 | `10.0.1.12` | Static Assets | Nginx serving CSS/JS/images on port 80 |
| Server 4 | `10.0.1.13` | File Storage | MinIO (S3-compatible) on port 9000 |

**The Reverse Proxy (Nginx):** Public IP `203.0.113.50`

None of the 4 backend servers are directly accessible from the internet. Only the Nginx reverse proxy has a public IP. The DNS record for `shop.example.com` points to `203.0.113.50`.

#### Step-by-Step: What Happens When a User Visits the Site

**Step 1 — DNS Resolution**

The user types `shop.example.com` in their browser. DNS resolves it to `203.0.113.50` — the reverse proxy, not any backend server.

**Step 2 — HTTPS Connection**

The browser establishes an HTTPS connection with `203.0.113.50`. The Nginx reverse proxy **terminates SSL** here. From this point, internal communication between Nginx and backend servers happens over plain HTTP on the private network. This offloads the encryption work from the backend servers.

**Step 3 — Request Routing**

The Nginx reverse proxy inspects the request URL and applies routing rules:

```nginx
upstream api_servers {
    # Load balance between Server 1 and Server 2
    server 10.0.1.10:8000;   # Server 1 - API Primary
    server 10.0.1.11:8000;   # Server 2 - API Replica
}

server {
    listen 443 ssl;
    server_name shop.example.com;

    ssl_certificate     /etc/ssl/shop.example.com.crt;
    ssl_certificate_key /etc/ssl/shop.example.com.key;

    # API requests → load balanced between Server 1 & 2
    location /api/ {
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Static files → Server 3
    location /static/ {
        proxy_pass http://10.0.1.12:80;
    }

    # Uploaded files → Server 4
    location /uploads/ {
        proxy_pass http://10.0.1.13:9000;
    }

    # Everything else → API servers
    location / {
        proxy_pass http://api_servers;
    }
}
```

**Step 4 — Routing in Action**

Let's trace 4 different requests from the same user:

| # | User's Request | Nginx Routes To | Why |
|---|---|---|---|
| 1 | `GET /api/products` | `10.0.1.10:8000` (Server 1) | Matches `/api/` → round-robin picks Server 1 |
| 2 | `GET /api/cart` | `10.0.1.11:8000` (Server 2) | Matches `/api/` → round-robin picks Server 2 |
| 3 | `GET /static/style.css` | `10.0.1.12:80` (Server 3) | Matches `/static/` → goes to static server |
| 4 | `GET /uploads/receipt.pdf` | `10.0.1.13:9000` (Server 4) | Matches `/uploads/` → goes to storage server |

**Step 5 — Response Returns**

Each backend server sends its response to the reverse proxy. The reverse proxy forwards it to the client. The client's browser sees **every response** coming from `203.0.113.50`. It has no idea that 4 different servers were involved.

#### What the Client Sees vs What Actually Happens

| | Client's View | Reality |
|---|---|---|
| **Server IP** | `203.0.113.50` | 4 different internal IPs |
| **Number of servers** | 1 | 4 |
| **SSL handled by** | `shop.example.com` | Nginx only (backends use plain HTTP) |
| **If Server 2 goes down** | Nothing changes | Nginx stops routing to it, Server 1 takes all API traffic |
| **Server internal ports** | Hidden | 8000, 80, 9000 |

This is the power of a reverse proxy: the complexity of 4 servers, different ports, different responsibilities, health checks, and failover is **completely invisible** to the client.

### Real-World Use Cases

| Use Case | Description |
|---|---|
| **SSL/TLS Termination** | The reverse proxy handles encryption/decryption (HTTPS) so backend servers deal only with plain HTTP, reducing their computational load. |
| **Caching** | Frequently requested static assets (images, CSS, JS) are cached at the reverse proxy layer, reducing load on origin servers. |
| **Compression** | The reverse proxy compresses responses (e.g., gzip/brotli) before sending them to clients. |
| **Security & DDoS Protection** | The origin servers' IPs are hidden. The reverse proxy can filter malicious traffic, rate-limit, and absorb DDoS attacks. |
| **Routing** | Route requests to different backend services based on URL path (e.g., `/api` → API servers, `/static` → CDN or file servers). |
| **A/B Testing** | Route a percentage of traffic to a new version of the application while the rest goes to the stable version. |

### Popular Reverse Proxies

- **Nginx** — The most widely used reverse proxy and web server.
- **HAProxy** — High-performance TCP/HTTP load balancer and reverse proxy.
- **Traefik** — Cloud-native reverse proxy, popular in Docker/Kubernetes environments.
- **Envoy** — Modern, high-performance proxy designed for service meshes.
- **AWS ALB / Cloudflare** — Cloud-managed reverse proxy services.

### Key Characteristics

- The **client does not know** about the proxy. It thinks it is talking to the real server.
- The **server knows** the proxy (they are deployed together in the same infrastructure).
- It protects and optimizes the **server side**.

---

## 4. Load Balancer

A **load balancer** distributes incoming network traffic across multiple servers to ensure no single server is overwhelmed.

`It is a **specialized form of reverse proxy** whose primary job is traffic distribution.`

![Load Balancer](/System_Design/images/load_balancer.svg)

### The Airport Check-In Analogy

You arrive at an airport with 10 check-in counters.

If everyone goes to Counter 1, the line would be impossibly long while the other 9 counters sit idle.

Instead, a **staff member** at the entrance directs passengers:

**Staff:** "Counter 3 is free, please go there."

Next person: **"Counter 7, please."**

This staff member is the **load balancer**. She distributes passengers (requests) across counters (servers) to ensure even utilization and minimum wait time.

### How It Works

1. All client requests arrive at the load balancer's IP address.
2. The load balancer selects a healthy backend server using a **load balancing algorithm**.
3. The request is forwarded to the selected server.
4. The server processes the request and sends the response back through the load balancer.
5. The load balancer forwards the response to the client.

### Load Balancing Algorithms

| Algorithm | How It Works |
|---|---|
| **Round Robin** | Distributes requests sequentially across servers. Server 1 → Server 2 → Server 3 → Server 1 → ... |
| **Weighted Round Robin** | Same as Round Robin, but servers with higher capacity get a higher weight (more requests). |
| **Least Connections** | Sends the request to the server with the fewest active connections. Best when requests have varying processing times. |
| **Weighted Least Connections** | Combines least connections with server weights. |
| **IP Hash** | Hashes the client's IP address to determine which server gets the request. Ensures the same client always hits the same server (sticky sessions). |
| **Least Response Time** | Routes to the server with the fastest response time and fewest active connections. |
| **Random** | Picks a server at random. Simple but surprisingly effective with large server pools. |

### Layer 4 vs Layer 7 Load Balancing

Load balancers operate at different layers of the OSI model:

| Feature | Layer 4 (Transport) | Layer 7 (Application) |
|---|---|---|
| **Operates on** | TCP/UDP packets | HTTP/HTTPS requests |
| **Routing decisions based on** | IP address, port number | URL path, headers, cookies, request body |
| **Speed** | Faster (less inspection) | Slower (deep packet inspection) |
| **Use case** | Simple traffic distribution | Content-based routing, A/B testing, API gateway |
| **Example** | AWS NLB | AWS ALB, Nginx, HAProxy |

### Health Checks

A load balancer continuously monitors the health of backend servers:

- **Active Health Checks:** The load balancer periodically sends a probe request (e.g., `GET /health`) to each server. If a server fails to respond, it is removed from the pool.
- **Passive Health Checks:** The load balancer monitors actual traffic. If a server starts returning errors or timing out, it is marked unhealthy.

If a server goes down, the load balancer automatically **stops sending traffic** to it and redistributes the load to healthy servers. When the server recovers, it is added back.

### Practical Example: Load Balancer with 4 Identical API Servers

Let's say you are running a high-traffic REST API for a food delivery platform. You have **4 identical Django servers** all running the same code, connected to the same database. An **HAProxy load balancer** sits in front of them.

![Load Balancer with 4 API Servers](/System_Design/images/load_balancer_4_servers.svg)

**Your infrastructure:**

| Server | Private IP | Specs | Weight | Status |
|---|---|---|---|---|
| Server 1 | `10.0.2.10` | 8 CPU, 32GB RAM | 3 (high capacity) | ✅ Healthy |
| Server 2 | `10.0.2.11` | 8 CPU, 32GB RAM | 3 (high capacity) | ✅ Healthy |
| Server 3 | `10.0.2.12` | 4 CPU, 16GB RAM | 2 (smaller) | ✅ Healthy |
| Server 4 | `10.0.2.13` | 4 CPU, 16GB RAM | 2 (deploying v2.1) | ❌ Down |

**The Load Balancer (HAProxy):** Public IP `203.0.113.100`

All 4 servers run the exact same Django application on port 8000. The only difference is their hardware capacity — Servers 1 & 2 are beefier machines, so they get a higher weight.

#### HAProxy Configuration

```haproxy
frontend http_front
    bind *:443 ssl crt /etc/ssl/api.foodapp.com.pem
    default_backend api_servers

backend api_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    server s1 10.0.2.10:8000 weight 3 check inter 5s fall 3 rise 2
    server s2 10.0.2.11:8000 weight 3 check inter 5s fall 3 rise 2
    server s3 10.0.2.12:8000 weight 2 check inter 5s fall 3 rise 2
    server s4 10.0.2.13:8000 weight 2 check inter 5s fall 3 rise 2
```

**What this means:**
- `balance roundrobin` — distribute sequentially, respecting weights.
- `weight 3` — Server 1 & 2 get 3 shares each; Server 3 & 4 get 2 shares each (total = 10, so S1 gets 30%, S2 gets 30%, S3 gets 20%, S4 gets 20%).
- `check inter 5s` — send a health probe every 5 seconds.
- `fall 3` — mark a server as down after 3 consecutive failures.
- `rise 2` — mark a recovered server as healthy after 2 consecutive successes.

#### Step-by-Step: What Happens When 10,000 Requests Hit the API

**Step 1 — DNS resolves to the load balancer**

`api.foodapp.com` resolves to `203.0.113.100` — the HAProxy load balancer, not any individual server.

**Step 2 — HAProxy terminates SSL**

The load balancer handles HTTPS. Internal traffic to the Django servers is plain HTTP.

**Step 3 — Health check detects Server 4 is down**

Server 4 is being redeployed with version 2.1. HAProxy's health check (`GET /health`) fails 3 times in a row → Server 4 is **automatically removed** from the pool.

**Step 4 — Traffic is distributed across healthy servers**

With Server 4 out, the weights are recalculated:

| Server | Original Share | After S4 Down |
|---|---|---|
| Server 1 | 30% (3/10) | 37.5% (3/8) |
| Server 2 | 30% (3/10) | 37.5% (3/8) |
| Server 3 | 20% (2/10) | 25% (2/8) |
| Server 4 | 20% (2/10) | 0% (removed) |

Of 10,000 requests per second:
- **Server 1** handles ~3,750 req/sec
- **Server 2** handles ~3,750 req/sec
- **Server 3** handles ~2,500 req/sec
- **Server 4** handles 0 (down)

**Step 5 — Tracing 10 consecutive requests**

| Request | Routed To | Why |
|---|---|---|
| 1 | Server 1 | Round robin, weight 3 |
| 2 | Server 2 | Round robin, weight 3 |
| 3 | Server 1 | S1 has higher weight, gets extra turn |
| 4 | Server 3 | Round robin, weight 2 |
| 5 | Server 2 | S2 has higher weight, gets extra turn |
| 6 | Server 1 | Back to S1 |
| 7 | Server 2 | Back to S2 |
| 8 | Server 3 | S3's turn |
| 9 | Server 1 | S1 again |
| 10 | Server 2 | S2 again |

Server 4 is **never hit**. Clients experience zero downtime.

**Step 6 — Server 4 finishes deployment and recovers**

Once Server 4's `/health` endpoint returns 200 for 2 consecutive checks → HAProxy **automatically adds it back** to the pool. Traffic distribution returns to the original 30/30/20/20 split.

#### What the Client Sees vs What Actually Happens

| | Client's View | Reality |
|---|---|---|
| **Server IP** | `203.0.113.100` | 4 different internal IPs |
| **Server count** | 1 | 4 (3 active, 1 recovering) |
| **When S4 went down** | Nothing changed | HAProxy auto-rerouted traffic in <5 seconds |
| **When S4 recovered** | Nothing changed | HAProxy auto-added it back |
| **Response time** | Consistent ~50ms | Load is evenly spread, no single server overwhelmed |
| **During deployment** | Zero downtime | Rolling deploy: take 1 server out, update, put back, repeat |

This is the power of a load balancer: it provides **high availability, horizontal scaling, and zero-downtime deployments** — all invisible to the client.

### Real-World Use Cases

| Use Case | Description |
|---|---|
| **Horizontal Scaling** | Distribute traffic across multiple identical servers to handle high load. |
| **High Availability** | If one server crashes, the others continue serving traffic seamlessly. |
| **Zero-Downtime Deployments** | Roll out updates by draining traffic from one server at a time, updating it, and adding it back. |
| **Geographic Distribution** | Route users to the nearest data center using global load balancing (e.g., AWS Route 53, Cloudflare). |
| **Microservices** | Route traffic to different services based on URL paths or headers. |

---

## 5. Comparison: Forward Proxy vs Reverse Proxy vs Load Balancer

![Forward Proxy vs Reverse Proxy vs Load Balancer](/System_Design/images/proxy_comparison.svg)

| Feature | Forward Proxy | Reverse Proxy | Load Balancer |
|---|---|---|---|
| **Sits in front of** | Clients | Servers | Servers |
| **Acts on behalf of** | Client | Server | Server |
| **Client awareness** | Client knows about the proxy | Client does NOT know | Client does NOT know |
| **Server awareness** | Server does NOT know | Server knows | Server knows |
| **Primary purpose** | Privacy, filtering, caching for clients | Security, caching, SSL termination for servers | Distributing traffic across multiple servers |
| **Hides identity of** | Client (client's IP hidden from server) | Server (server's IP hidden from client) | Server (server's IP hidden from client) |
| **Traffic direction** | Outbound (client → internet) | Inbound (internet → server) | Inbound (internet → servers) |
| **Number of backend servers** | Usually targets many external servers | One or more backend servers | Multiple backend servers (2+) |
| **Caching** | Yes | Yes | Sometimes (Layer 7) |
| **SSL Termination** | Rarely | Yes | Yes (Layer 7) |
| **Health Checks** | No | Sometimes | Yes (core feature) |
| **Load Distribution** | No | Optional | Yes (core feature) |
| **Example Tools** | Squid, corporate firewalls, VPNs | Nginx, Traefik, Envoy, Cloudflare | HAProxy, AWS ALB/NLB, Nginx |

### The Relationship Between Them

```
Forward Proxy                Reverse Proxy              Load Balancer
     │                            │                          │
     │                            │                          │
  Protects                    Protects                   Distributes
  CLIENTS                     SERVERS                    TRAFFIC
     │                            │                          │
     │                            ├──────────────────────────┤
     │                            │                          │
     │                       A Load Balancer IS a specialized
     │                       type of Reverse Proxy
     │
  Completely different direction
  (outbound vs inbound)
```

> **Key Insight:** Every load balancer is essentially a reverse proxy, but not every reverse proxy is a load balancer. A reverse proxy can have just one backend server and still provide value (SSL termination, caching, security). A load balancer's core value proposition is distributing traffic across **multiple** servers.

---

## 6. Can They Work Together?

Absolutely. In a production architecture, you often see **all of them** working together:

```
                                          ┌──► App Server 1
Corporate     Forward      Reverse Proxy  │
Network   ──► Proxy   ──►  (Nginx +    ──►├──► App Server 2
(Clients)    (Squid)      Load Balancer)  │
                                          └──► App Server 3
```

**Step-by-step flow:**

1. A client inside a corporate network makes a request.
2. The **forward proxy** (e.g., Squid) intercepts the request, checks if the destination is allowed, logs the request, and forwards it.
3. The request hits the **reverse proxy** (e.g., Nginx) sitting at the edge of the server infrastructure.
4. Nginx terminates SSL, checks cache, applies rate limiting, and then uses its **load balancing** module to pick one of the three backend app servers.
5. The chosen app server processes the request and returns the response through the same chain.

---

## 7. Quick Decision Guide

**When do I need a forward proxy?**
- You want to control, monitor, or restrict **outbound** traffic from your network.
- You need to anonymize client identities from external servers.
- You want to cache external resources for internal users.

**When do I need a reverse proxy?**
- You want to hide your backend infrastructure from the public internet.
- You need SSL termination, caching, or compression at the edge.
- You want to route requests to different backend services based on URL patterns.

**When do I need a load balancer?**
- You have **multiple instances** of the same service and need to distribute traffic.
- You need high availability and fault tolerance.
- You need health checks and automatic failover.

**When do I need both a reverse proxy and a load balancer?**
- In most production systems, they are the **same component**. Nginx and HAProxy both act as reverse proxies **and** load balancers simultaneously.

---

## Summary

| Concept | One-Liner |
|---|---|
| **Proxy** | A middleman between two parties in a network. |
| **Forward Proxy** | Sits in front of clients, hides client identity, controls outbound traffic. |
| **Reverse Proxy** | Sits in front of servers, hides server identity, controls inbound traffic. |
| **Load Balancer** | A reverse proxy specialized in distributing traffic across multiple servers. |
