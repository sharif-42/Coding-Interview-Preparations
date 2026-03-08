# Introduction to Kubernetes(K8s)

### First: The problem Kubernetes solves

Imagine we built a Django app. To run it, we need:
- Python
- Django
- Dependencies
- Database connection
- Environment variables

We package everything into a container using Docker. Good. Now our app runs anywhere.

But real production is messy. We might need:

- 10 copies of our app running (for traffic)
- Restart app automatically if it crashes
- Scale up when traffic increases
- Deploy new version without downtime
- Run across multiple servers
- Managing this manually is extremely difficult.

This is exactly why Kubernetes exists.

### The journey without Kubernetes vs with Kubernetes

**Without Kubernetes (manual approach):**
```
1. SSH into Server A → docker run my-app
2. SSH into Server B → docker run my-app
3. Traffic spike → SSH into Server C → docker run my-app
4. Server A crashes → SSH into Server D → docker run my-app
5. Need to update → SSH into every server → docker stop, docker pull, docker run
6. Something breaks at 3 AM → you get a phone call
```

**With Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3     # Kubernetes ensures 3 copies always running
  ...
```
```
- Server crashes? Kubernetes restarts on another server. Automatically.
- Traffic spike? Kubernetes adds more copies. Automatically.
- New version? Kubernetes rolls out gradually. Zero downtime.
- 3 AM? Kubernetes handles it. You sleep.
```

### Brief history

| Year | Event |
| --- | --- |
| 2003-2004 | Google builds **Borg** — internal container orchestration system |
| 2013 | Google builds **Omega** — next-gen of Borg |
| 2014 | Google open-sources **Kubernetes** (based on lessons from Borg and Omega) |
| 2015 | Kubernetes v1.0 released, donated to the newly formed **CNCF** (Cloud Native Computing Foundation) |
| 2018+ | Kubernetes becomes the industry standard for container orchestration |
| Today | Supported by all major cloud providers (AWS EKS, Azure AKS, Google GKE) |

The name "Kubernetes" comes from Greek — meaning **helmsman** or **pilot**. The K8s abbreviation replaces the 8 letters between "K" and "s".


## Simple definition
`Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.`

**What Kubernetes guarantees:**
- The apps are running
- Correct number of copies exist
- Crashed apps restart automatically
- Traffic is distributed properly
- Scaling happens automatically

Kubernetes is often abbreviated as **K8s** (K + 8 letters + s).

### Key features at a glance

| Feature | What it means | Example |
| --- | --- | --- |
| **Self-healing** | Automatically restarts failed containers | Pod crashes → new one starts in seconds |
| **Auto-scaling** | Adjusts replicas based on load | CPU hits 80% → scale from 3 to 6 Pods |
| **Rolling updates** | Deploys new versions without downtime | v1 → v2 gradually, one Pod at a time |
| **Rollbacks** | Reverts to previous version if something breaks | v2 has bugs → auto-rollback to v1 |
| **Service discovery** | Apps find each other by name, not IP | `my-api.default.svc.cluster.local` |
| **Load balancing** | Distributes traffic across Pods | 1000 requests → spread across 5 Pods |
| **Secret management** | Stores sensitive data securely | DB passwords stored as Kubernetes Secrets |
| **Storage orchestration** | Mounts persistent storage to Pods | Attach AWS EBS volume to a database Pod |
| **Declarative config** | You describe *what* you want, K8s figures out *how* | "I want 3 nginx Pods" → done |

### Declarative vs Imperative

Kubernetes uses a **declarative model** — this is a core philosophy.

**Imperative** (telling the system *what to do* step by step):
```bash
kubectl run nginx --image=nginx        # create a pod
kubectl scale deployment nginx --replicas=3  # manually scale
kubectl delete pod nginx-abc123        # manually delete
```

**Declarative** (telling the system *what you want*, it figures out the steps):
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```
```bash
kubectl apply -f deployment.yaml   # apply desired state
# Kubernetes handles creating, updating, or deleting resources to match
```

**Why declarative is better:**
- Config files can be version-controlled (Git)
- Reproducible across environments (dev, staging, prod)
- Kubernetes continuously reconciles actual state → desired state
- Audit trail of changes


### Real-life analogy: Restaurant manager

Think of containers as cooks. Kubernetes is the restaurant manager. Manager’s job:

- Make sure enough cooks are working
- Replace sick cooks immediately
- Assign orders evenly
- Add more cooks during rush hour
- Remove extra cooks when quiet

You don’t manage cooks manually. Manager handles everything. `Kubernetes is that manager`.
**Extended analogy mapping:**

| Restaurant | Kubernetes |
| --- | --- |
| Restaurant chain | Cluster |
| Individual restaurant | Node |
| Kitchen station | Pod |
| Cook | Container |
| Recipe | Container Image |
| Menu | Deployment spec |
| Manager | Control Plane |
| Order distribution | Load Balancing |
| "We need 5 cooks tonight" | `replicas: 5` |
| "Replace that sick cook" | Self-healing |

## Core building blocks (simple explanation)

### 1. Container

Your app packaged with everything needed. Example: Django app inside Docker container.

A container includes:
- Your application code
- Runtime (Python, Node.js, etc.)
- System libraries
- Dependencies

```
┌──────────────────────┐
│     Container        │
│  ┌────────────────┐  │
│  │  Your App Code │  │
│  ├────────────────┤  │
│  │  Dependencies  │  │
│  ├────────────────┤  │
│  │    Runtime     │  │
│  ├────────────────┤  │
│  │   OS (slim)    │  │
│  └────────────────┘  │
└──────────────────────┘
```

### 2. Pod

Smallest deployable unit in Kubernetes.

Pod = container runner. Think: Pod → runs your container.

**Key facts about Pods:**
- A Pod runs **one or more containers** (usually just one)
- All containers in a Pod share the same **network** (localhost) and **storage**
- Pods are **ephemeral** — they can be killed and recreated at any time
- Pods get their own **IP address**
- You almost never create Pods directly — you use Deployments

**Single container Pod (most common):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-django-app
  labels:
    app: django
spec:
  containers:
  - name: django
    image: my-django-app:1.0
    ports:
    - containerPort: 8000
```

**Multi-container Pod (sidecar pattern):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging
spec:
  containers:
  - name: app
    image: my-app:1.0
  - name: log-collector        # sidecar container
    image: fluentd:latest
```

When would you use multiple containers in a Pod?
- **Sidecar**: log collector, monitoring agent
- **Ambassador**: proxy for external service
- **Adapter**: transforms output format for external consumers

### 3. Node
A server (machine) that runs Pods. Example:

- DigitalOcean Droplet
- AWS EC2 instance
- Azure VM
- Your local VM (minikube)

**Two types of nodes:**
- **Control Plane Node** — runs the brain components (API Server, Scheduler, etc.)
- **Worker Node** — runs your application Pods

Each worker node has:
- **Kubelet** — agent that manages Pods
- **Kube-proxy** — manages networking
- **Container runtime** — runs containers (containerd, CRI-O)

```bash
# View all nodes
kubectl get nodes

# Example output:
# NAME           STATUS   ROLES           AGE   VERSION
# control-plane  Ready    control-plane   30d   v1.28.0
# worker-1       Ready    <none>          30d   v1.28.0
# worker-2       Ready    <none>          30d   v1.28.0
```

### 4. Cluster
Group of nodes working together.

Cluster = Control Plane + Worker Nodes

Kubernetes manages the cluster. A cluster can range from a single node (minikube for development) to thousands of nodes (Google runs 15,000+ node clusters).

```
┌──────────────── Cluster ─────────────────┐
│                                          │
│  ┌──────────────────┐                    │
│  │  Control Plane   │                    │
│  └────────┬─────────┘                    │
│           │                              │
│     ┌─────┼─────┐                        │
│     ▼     ▼     ▼                        │
│  ┌─────┐┌─────┐┌─────┐                   │
│  │Node1││Node2││Node3│  ← Worker Nodes   │
│  │[Pod]││[Pod]││[Pod]│                    │
│  │[Pod]││[Pod]││     │                    │
│  └─────┘└─────┘└─────┘                   │
└──────────────────────────────────────────┘
```

### 5. Deployment

A Deployment manages the lifecycle of your Pods. It defines:

- Which app to run (container image)
- How many copies (replicas)
- How to update (rolling update strategy)
- How to rollback

**Example: Deploy 3 copies of a Django app**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # max 1 extra Pod during update
      maxUnavailable: 0    # never have fewer than 3 running
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
      - name: django
        image: my-django-app:2.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

**Deployment hierarchy:**
```
Deployment
  └── ReplicaSet (managed automatically)
        ├── Pod 1
        ├── Pod 2
        └── Pod 3
```

**Common Deployment commands:**
```bash
# Create or update
kubectl apply -f deployment.yaml

# Check status
kubectl rollout status deployment/django-app

# View history
kubectl rollout history deployment/django-app

# Rollback to previous version
kubectl rollout undo deployment/django-app

# Scale manually
kubectl scale deployment django-app --replicas=5
```

### 6. Service

Provides a **stable network endpoint** to access your Pods.

**Why do we need Services?** Pods are ephemeral — they get new IPs when recreated. A Service gives a permanent IP and DNS name.

```
                  ┌────────────────┐
   Users  ──────►│    Service     │
                  │  10.96.0.50    │
                  │  django-svc    │
                  └───────┬────────┘
                    ┌─────┼─────┐
                    ▼     ▼     ▼
                 Pod 1  Pod 2  Pod 3
               (dynamic IPs — don't matter)
```

**Service types:**

| Type | What it does | Use case |
| --- | --- | --- |
| **ClusterIP** (default) | Internal IP only, accessible within cluster | Backend API used by other services |
| **NodePort** | Exposes service on each node's IP at a static port (30000-32767) | Development, quick testing |
| **LoadBalancer** | Creates an external cloud load balancer | Production web apps |
| **ExternalName** | Maps service to an external DNS name | Accessing external databases |

**Example: ClusterIP Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  type: ClusterIP
  selector:
    app: django        # matches Pods with label app=django
  ports:
  - port: 80           # Service listens on port 80
    targetPort: 8000   # forwards to Pod's port 8000
```

```bash
# Other Pods can now access it via:
# http://django-service.default.svc.cluster.local:80
# or simply:
# http://django-service:80  (within the same namespace)
```

### 7. Namespace

A way to divide cluster resources into isolated groups. Think of it as **folders for your Kubernetes objects**.

```bash
# Default namespaces in every cluster:
# default           — where your stuff goes if you don't specify
# kube-system       — Kubernetes system components
# kube-public       — publicly accessible data
# kube-node-lease   — node heartbeat data
```

**Why use namespaces?**
- Separate environments: `dev`, `staging`, `production`
- Separate teams: `team-frontend`, `team-backend`
- Resource quotas: limit CPU/memory per namespace
- Access control: different permissions per namespace

```bash
# Create a namespace
kubectl create namespace staging

# Deploy to a specific namespace
kubectl apply -f deployment.yaml -n staging

# List Pods in a namespace
kubectl get pods -n staging

# List Pods in all namespaces
kubectl get pods --all-namespaces
```

### 8. ConfigMap and Secret

**ConfigMap** — stores non-sensitive configuration as key-value pairs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  DEBUG: "false"
  ALLOWED_HOSTS: "*"
```

**Secret** — stores sensitive data (passwords, tokens, keys). Base64 encoded (not encrypted by default!).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64 encoded "password123"
  DB_USER: YWRtaW4=                # base64 encoded "admin"
```

**Using them in a Pod:**
```yaml
spec:
  containers:
  - name: app
    image: my-app:1.0
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-credentials
```

### 9. Ingress

Manages external HTTP/HTTPS access to your Services. Like a **smart reverse proxy**.

```
           Internet
              │
     ┌────────▼─────────┐
     │  Ingress Controller│ (NGINX, Traefik, etc.)
     └────────┬─────────┘
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
  /api     /web       /admin
    │         │          │
  Service   Service   Service
  (API)    (Frontend) (Admin)
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 10. Volumes and Persistent Storage

Containers are **ephemeral** — when they restart, all data inside is lost. Volumes solve this.

**Types of volumes:**

| Volume Type | Persistence | Use Case |
| --- | --- | --- |
| `emptyDir` | Deleted when Pod dies | Temporary cache, shared data between containers in a Pod |
| `hostPath` | Data lives on the node | Single-node dev/testing only |
| `PersistentVolume` (PV) | Independent of Pod lifecycle | Databases, file uploads |
| `PersistentVolumeClaim` (PVC) | Request for storage | App requests "give me 10Gi of storage" |

**PersistentVolume workflow:**
```
 Admin creates PV          Developer creates PVC          Pod uses PVC
 (available storage)  →    (request for storage)     →   (mounted as a directory)

 PV: "I have 50Gi SSD"    PVC: "I need 10Gi"            Pod: /data → PVC
```

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
# Using it in a Pod
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-data
```

---

## How Objects Relate to Each Other

```
Cluster
  ├── Namespace ("default")
  │     ├── Deployment
  │     │     └── ReplicaSet
  │     │           ├── Pod → Container(s)
  │     │           ├── Pod → Container(s)
  │     │           └── Pod → Container(s)
  │     ├── Service → selects Pods by labels
  │     ├── Ingress → routes to Services
  │     ├── ConfigMap → injected into Pods
  │     ├── Secret → injected into Pods
  │     └── PVC → mounted into Pods
  │
  ├── Namespace ("kube-system")
  │     ├── CoreDNS Pods
  │     ├── kube-proxy DaemonSet
  │     └── Metrics Server
  │
  └── Nodes
        ├── Worker Node 1 (runs Pods)
        ├── Worker Node 2 (runs Pods)
        └── Control Plane Node (runs K8s system)
```

---

## Labels and Selectors

Labels are key-value pairs attached to objects. Selectors are how you find objects by their labels. This is how Kubernetes **connects things together**.

```yaml
# Pod with labels
metadata:
  labels:
    app: django
    environment: production
    version: v2
```

```yaml
# Service selects Pods by label
spec:
  selector:
    app: django          # "find all Pods with label app=django"
```

```bash
# Query by labels
kubectl get pods -l app=django
kubectl get pods -l environment=production
kubectl get pods -l "app=django,version=v2"
```

---

## Getting Started: Local Kubernetes Setup

You don't need a cloud account to learn Kubernetes. These tools run a cluster on your laptop:

| Tool | Best for | Notes |
| --- | --- | --- |
| **minikube** | Learning, development | Single-node cluster in a VM or container |
| **kind** (Kubernetes in Docker) | CI/CD, testing | Runs K8s nodes as Docker containers |
| **k3s** | Lightweight production | Minimal K8s distribution by Rancher |
| **Docker Desktop** | Mac/Windows developers | Built-in K8s toggle |

**Quick start with minikube:**
```bash
# Install minikube (Linux)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a cluster
minikube start

# Verify it works
kubectl get nodes
kubectl get pods -A

# Deploy something
kubectl create deployment hello --image=nginx
kubectl expose deployment hello --port=80 --type=NodePort
minikube service hello   # opens in browser

# Clean up
minikube delete
```

---

## Essential kubectl Commands

```bash
# ─── Cluster Info ───
kubectl cluster-info
kubectl get nodes

# ─── Pods ───
kubectl get pods                     # list pods
kubectl get pods -o wide             # show node, IP details
kubectl describe pod <name>          # detailed info
kubectl logs <pod-name>              # view logs
kubectl logs <pod-name> -f           # follow logs (live)
kubectl exec -it <pod-name> -- bash  # shell into a pod

# ─── Deployments ───
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl scale deployment/<name> --replicas=5

# ─── Services ───
kubectl get services
kubectl expose deployment <name> --port=80 --type=NodePort

# ─── Debugging ───
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod <name>          # check Events section
kubectl top pods                     # CPU/memory usage
kubectl top nodes

# ─── Cleanup ───
kubectl delete -f deployment.yaml
kubectl delete pod <name>
kubectl delete deployment <name>
```

---

## Common Interview Questions

### Q: What is Kubernetes and why do we need it?
Kubernetes is an open-source container orchestration platform. We need it because managing containers manually across multiple servers is error-prone, doesn't scale, and can't self-heal. Kubernetes automates deployment, scaling, load balancing, and self-healing of containerized applications.

### Q: What is the difference between Docker and Kubernetes?
| Docker | Kubernetes |
| --- | --- |
| Builds and runs containers | Orchestrates and manages containers |
| Runs on a single machine | Runs across a cluster of machines |
| No auto-healing | Auto-restarts failed containers |
| Manual scaling | Auto-scaling |
| No built-in load balancing | Built-in load balancing |

They're **complementary**, not competing. Docker creates containers, Kubernetes manages them at scale.

### Q: What is a Pod? Why not just use containers directly?
A Pod is the smallest deployable unit in Kubernetes — a wrapper around one or more containers. Kubernetes manages Pods (not containers) because:
- Pods provide shared networking (containers in a Pod share localhost)
- Pods provide shared storage (volumes)
- Pods enable sidecar patterns (logging, monitoring containers)
- Pods are the unit of scheduling, scaling, and replication

### Q: What is the difference between a Deployment and a Pod?
- **Pod**: A single instance of your app. If it dies, it's gone.
- **Deployment**: Manages Pods — ensures the desired number are running, handles updates and rollbacks. Always use Deployments in production.

### Q: What happens when you run `kubectl apply -f deployment.yaml`?
1. `kubectl` sends the YAML to the API Server
2. API Server validates and stores it in etcd
3. Deployment Controller creates a ReplicaSet
4. ReplicaSet Controller creates Pod objects
5. Scheduler assigns each Pod to a node
6. Kubelet on each node pulls the image and starts containers

### Q: What is the difference between ClusterIP, NodePort, and LoadBalancer?
- **ClusterIP**: Internal only. Other Pods in the cluster can reach it.
- **NodePort**: Opens a port (30000-32767) on every node. External access via `NodeIP:NodePort`.
- **LoadBalancer**: Provisions a cloud load balancer with a public IP. Best for production.

### Q: What are namespaces used for?
Namespaces provide logical isolation within a cluster. Used for separating environments (dev/staging/prod), teams, or projects. They enable resource quotas and access control per group.

### Q: How is Kubernetes different from Docker Compose?
| Docker Compose | Kubernetes |
| --- | --- |
| Single host only | Multi-host cluster |
| No self-healing | Auto-restarts, auto-scales |
| Dev/testing | Production-ready |
| Simple YAML | More complex, more powerful |
| `docker-compose up` | `kubectl apply -f` |

---

## References
1. [Kubernetes Official Documentation](https://kubernetes.io/docs/home/)
2. [What is Kubernetes? — Kubernetes.io](https://kubernetes.io/docs/concepts/overview/)
3. [Kubernetes Basics Tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
4. [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
5. [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)