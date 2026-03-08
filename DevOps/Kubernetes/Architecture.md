# Kubernetes Architecture
A Kubernetes cluster consists of a control plane plus a set of worker machines, called nodes, that run containerized applications. Every cluster needs at least one worker node in order to run Pods.

The worker node(s) host the Pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster. In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.

Kubernetes has two main parts:
1. Control Plane (Brain)
2. Worker Nodes (Workers)

### High-Level Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
│                                                                  │
│  ┌────────────────────── CONTROL PLANE ───────────────────────┐  │
│  │                                                            │  │
│  │   ┌──────────┐  ┌───────────┐  ┌────────────────────────┐  │  │
│  │   │   etcd   │  │ Scheduler │  │  Controller Manager    │  │  │
│  │   └────┬─────┘  └─────┬─────┘  └───────────┬────────────┘  │  │
│  │        │              │                     │              │  │
│  │        └──────────────┼─────────────────────┘              │  │
│  │                       │                                    │  │
│  │                ┌──────┴──────┐                             │  │
│  │                │  API Server │ ◄──── kubectl / Dashboard   │  │
│  │                └──────┬──────┘                             │  │
│  └───────────────────────┼────────────────────────────────────┘  │
│                          │                                       │
│          ┌───────────────┼───────────────┐                       │
│          │               │               │                       │
│  ┌───────┴────┐  ┌───────┴────┐  ┌───────┴────┐                 │
│  │  Worker 1  │  │  Worker 2  │  │  Worker 3  │                 │
│  │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │                 │
│  │ │kubelet │ │  │ │kubelet │ │  │ │kubelet │ │                 │
│  │ │kube-   │ │  │ │kube-   │ │  │ │kube-   │ │                 │
│  │ │proxy   │ │  │ │proxy   │ │  │ │proxy   │ │                 │
│  │ │runtime │ │  │ │runtime │ │  │ │runtime │ │                 │
│  │ ├────────┤ │  │ ├────────┤ │  │ ├────────┤ │                 │
│  │ │ Pod(s) │ │  │ │ Pod(s) │ │  │ │ Pod(s) │ │                 │
│  │ └────────┘ │  │ └────────┘ │  │ └────────┘ │                 │
│  └────────────┘  └────────────┘  └────────────┘                 │
└──────────────────────────────────────────────────────────────────┘
```

### Real life example: Food delivery company

Imagine a food delivery company.

- Head Office → makes decisions
- Delivery centers → deliver food
- Delivery workers → actually deliver food

Mapping to Kubernetes:

| Real life       | Kubernetes    |
| --------------- | ------------- |
| Head office     | Control Plane |
| Delivery center | Node          |
| Delivery worker | Pod           |
| Food package    | Container     |

## Control Plane (The Brain)
Control plane makes ALL decisions. It decides:

- Which server runs your app
- When to start containers
- When to restart failed containers
- When to scale up/down

Control plane has 4 main components:

### API Server (The Entry Gate)
This is the front door of Kubernetes. Everything talks through API Server. It is the **only component** that directly communicates with `etcd`.

Real life analogy: `Head office receptionist`. You say:

`Run 3 copies of my Django app`

Request goes to API Server first.

Technical name: `kube-apiserver`

**Key responsibilities:**
- Exposes the Kubernetes API (RESTful interface over HTTPS)
- Authenticates and authorizes all API requests
- Validates and processes API objects (Pods, Services, Deployments, etc.)
- Acts as the gateway to `etcd` — no other component reads/writes etcd directly
- Serves as the communication hub between all control plane components

**How it works step-by-step:**
```
User runs: kubectl apply -f deployment.yaml

1. kubectl sends HTTPS request → API Server
2. API Server authenticates the user (who are you?)
3. API Server authorizes the request (are you allowed?)
4. API Server validates the object (is the YAML correct?)
5. API Server writes to etcd (persist the desired state)
6. API Server notifies Scheduler and Controllers (via watch mechanism)
```

**Interview tip:** `The API Server is stateless — all state lives in etcd. You can run multiple API Server instances for high availability.`

### Scheduler (The Decision Maker)
Scheduler decides: Which server should run your app?

Example: You have 3 servers:
- Server A → busy
- Server B → free
- Server C → medium load

Scheduler picks Server B.

Real life analogy: `Manager assigning delivery worker to location.`

Technical name: `kube-scheduler`

**Scheduling process (2 phases):**

1. **Filtering** — Eliminates nodes that can't run the Pod
   - Not enough CPU/memory?
   - Node has a taint the Pod doesn't tolerate?
   - Node selector doesn't match?
   - Node is in `NotReady` state?

2. **Scoring** — Ranks the remaining nodes
   - Which node has the most resources available?
   - Does the node already have the container image cached?
   - Would scheduling here spread Pods across failure zones?

**Factors the Scheduler considers:**
| Factor | Example |
| --- | --- |
| Resource requests | Pod needs 512Mi memory, node has 1Gi free |
| Node affinity | "Run on nodes with SSD" |
| Pod anti-affinity | "Don't put 2 replicas on the same node" |
| Taints & tolerations | "This node is reserved for GPU workloads" |
| Topology spread | "Spread Pods across availability zones" |

**Important:** The Scheduler only *decides* where to place the Pod. It does NOT launch the Pod. It writes the node assignment to the API Server, and the `kubelet` on that node does the actual work.

### Controller Manager (The Supervisor)
This watches everything and fixes problems.
You want: `3 containers running` but `Only 2 running`

Controller detects problem and starts 1 more automatically.

Real life analogy: `Supervisor ensuring enough workers are present.`

Technical name: `kube-controller-manager`

**Core concept: The Reconciliation Loop**

Every controller follows the same pattern:
```
loop forever:
    desired_state = read from API Server (what user wants)
    current_state = observe actual cluster state
    if current_state != desired_state:
        take action to move current → desired
```

This is the heart of Kubernetes' **declarative model** — you declare *what* you want, controllers figure out *how* to achieve it.

**Important built-in controllers:**

| Controller | What it does |
| --- | --- |
| **ReplicaSet Controller** | Ensures the correct number of Pod replicas are running |
| **Deployment Controller** | Manages rolling updates and rollbacks |
| **Node Controller** | Monitors node health, marks unreachable nodes |
| **Job Controller** | Creates Pods to run tasks to completion |
| **Service Account Controller** | Creates default service accounts for new namespaces |
| **Endpoint Controller** | Populates endpoint objects (connects Services to Pods) |
| **CronJob Controller** | Creates Jobs on a schedule |

**Example scenario:**
```
1. You deploy: replicas: 3
2. ReplicaSet Controller sees: desired=3, current=0
3. It creates 3 Pod objects via API Server
4. Scheduler assigns each Pod to a node
5. Kubelet on each node starts the containers
6. If a Pod crashes → current=2, desired=3 → Controller creates 1 more
```

### Cloud Controller Manager (Cloud Integration)

If you run Kubernetes on a cloud provider (AWS, GCP, Azure), this component handles cloud-specific logic.

Technical name: `cloud-controller-manager`

**What it manages:**
- **Node Controller** — checks if a cloud VM still exists after it stops responding
- **Route Controller** — sets up network routes in the cloud infrastructure
- **Service Controller** — creates/updates/deletes cloud load balancers when you create a `Service` of type `LoadBalancer`

This separation allows the core Kubernetes code to remain cloud-agnostic.

### etcd (The Database)

Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data. This stores ALL Kubernetes information. Stores:
- Which apps are running
- How many containers
- Configuration
- Cluster state

Real life analogy: `Company records database.`

Technical name: `etcd`

**Key characteristics:**
- **Distributed** — runs across multiple nodes for fault tolerance (typically 3 or 5 instances)
- **Consistent** — uses the Raft consensus algorithm to ensure all nodes agree on data
- **Key-value store** — data is stored as key-value pairs, not tables
- **Watch support** — components can subscribe to changes (this is how Kubernetes stays reactive)

**What's stored in etcd (examples):**
```
/registry/pods/default/my-django-app-pod-abc123
/registry/services/default/my-service
/registry/deployments/default/my-deployment
/registry/secrets/default/db-password
/registry/configmaps/default/app-config
/registry/nodes/worker-node-1
```

**Critical rules:**
- Only the API Server communicates with etcd directly
- Always back up etcd — losing it means losing your entire cluster state
- etcd should run on fast SSDs for performance (it's I/O sensitive)
- In production, use an odd number of etcd nodes (3 or 5) for proper quorum

**Interview tip:** If etcd goes down, the cluster continues running existing workloads (kubelet keeps containers alive), but you cannot make any changes — no new deployments, no scaling, no updates.

## Worker Nodes (The Workers)

Worker nodes actually run your applications. Each worker node has 3 components:

### Kubelet (Node Manager)

An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod. Kubelet talks to control plane.

Real life analogy: Delivery center manager.

Technical name: `kubelet`

**Key responsibilities:**
- Registers the node with the API Server
- Watches the API Server for Pods assigned to its node
- Pulls container images and starts/stops containers via the Container Runtime
- Reports node and Pod status back to the API Server
- Executes liveness, readiness, and startup probes
- Mounts volumes, secrets, and configmaps into Pods

**How kubelet works:**
```
1. API Server assigns Pod to this node
2. Kubelet receives PodSpec (desired Pod definition)
3. Kubelet instructs Container Runtime to pull image
4. Container Runtime creates and starts the container
5. Kubelet continuously monitors container health
6. Kubelet reports status back to API Server every 10s (default)
```

**Important:** Kubelet does NOT manage containers that are not created by Kubernetes. If you manually run `docker run` on a node, kubelet ignores that container.

### Container Runtime (Runs containers)

A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment. This actually runs containers.

This runs your Django app container.

Real life analogy: Kitchen where food is cooked.

**Supported container runtimes:**

| Runtime | Description |
| --- | --- |
| **containerd** | Industry standard, lightweight. Default in most K8s distributions |
| **CRI-O** | Lightweight runtime built specifically for Kubernetes |
| **Docker Engine** | Removed as a direct runtime in K8s v1.24+ (but Docker images still work!) |

**CRI (Container Runtime Interface):**

Kubernetes doesn't talk to container runtimes directly. It uses a standard interface called **CRI**. Any runtime that implements CRI can work with Kubernetes.

```
Kubelet  ──CRI──►  containerd  ──►  runc  ──►  Linux Container
```

**Interview tip:** Docker was deprecated as a runtime in Kubernetes v1.24, but this does NOT mean Docker images stopped working. Images built with `docker build` follow the OCI standard and work with any CRI-compatible runtime.

### Kube Proxy (Network Manager)

kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept. Handles network communication. Ensures traffic reaches correct container.

Real life analogy: `Traffic controller directing delivery routes.`

Technical name: `kube-proxy`

**How kube-proxy works:**

When you create a Kubernetes `Service`, kube-proxy makes sure traffic to that Service reaches the right Pods — even if Pods move between nodes.

**Proxy modes:**

| Mode | How it works | Performance |
| --- | --- | --- |
| **iptables** (default) | Creates iptables rules for routing | Good for most clusters |
| **IPVS** | Uses Linux kernel IPVS for load balancing | Better for 1000+ Services |
| **nftables** | Uses nftables (newer iptables replacement) | Newer, available since K8s v1.29 |

**Example flow:**
```
User request → Service IP (10.96.0.1:80)
                  │
         kube-proxy routes to one of:
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  Pod A:8080   Pod B:8080   Pod C:8080
  (Node 1)    (Node 2)     (Node 1)
```

---

## How Components Communicate

Understanding the communication flow is crucial for interviews.

### What happens when you run `kubectl apply -f deployment.yaml`?

```
Step 1: kubectl → API Server
        "I want 3 replicas of nginx"

Step 2: API Server → etcd
        Stores the Deployment object

Step 3: Deployment Controller (watching API Server) detects new Deployment
        Creates a ReplicaSet object → API Server → etcd

Step 4: ReplicaSet Controller detects new ReplicaSet
        Creates 3 Pod objects → API Server → etcd
        (Pods are in "Pending" state, no node assigned yet)

Step 5: Scheduler detects unscheduled Pods
        Evaluates nodes, picks best fit for each Pod
        Updates Pod with node assignment → API Server → etcd

Step 6: Kubelet on each assigned node detects new Pod
        Pulls container image via Container Runtime
        Starts the container
        Reports Pod status as "Running" → API Server → etcd

Step 7: kube-proxy updates network rules
        If a Service exists, traffic can now reach the new Pods
```

### What happens when a Pod crashes?

```
Step 1: Kubelet detects container exited (via health checks or process exit)
Step 2: Kubelet attempts container restart (based on restartPolicy)
Step 3: If Pod itself is deleted/evicted:
        → ReplicaSet Controller sees: desired=3, current=2
        → Creates a new Pod object
        → Scheduler assigns it to a node
        → Kubelet starts it
```

### What happens when a Node fails?

```
Step 1: Kubelet stops sending heartbeats to API Server
Step 2: Node Controller (in Controller Manager) waits for timeout (default: 40s)
Step 3: Node Controller marks node as "NotReady"
Step 4: After pod-eviction-timeout (default: 5min):
        → Node Controller evicts all Pods from the failed node
        → ReplicaSet Controllers detect missing Pods
        → New Pods are scheduled on healthy nodes
```

---

## Addons (Important Extensions)

Addons are not part of the core architecture, but they are almost always present in real clusters.

### DNS (CoreDNS)
Every Kubernetes cluster should have cluster DNS. Pods use it to discover Services by name instead of IP.

```
Instead of: curl http://10.96.0.15:80
You can:    curl http://my-service.default.svc.cluster.local
```

### Ingress Controller
Manages external HTTP/HTTPS access to Services. Without it, Services are only accessible inside the cluster.

```
                    Internet
                       │
              ┌────────┴────────┐
              │ Ingress Controller│ (NGINX, Traefik, etc.)
              └────────┬────────┘
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   /api → Service A   /web → Service B   /admin → Service C
```

### Dashboard
Web-based UI for managing the cluster visually.

### Metrics Server
Collects resource metrics (CPU, memory) from kubelets. Required for `kubectl top` and Horizontal Pod Autoscaler.

---

## Kubernetes Networking Model

Kubernetes imposes these fundamental networking rules:

1. **Every Pod gets its own IP address** — no need for NAT between Pods
2. **All Pods can communicate with all other Pods** across nodes without NAT
3. **All Nodes can communicate with all Pods** without NAT
4. **The IP that a Pod sees itself as** is the same IP others see it as

**Network types in Kubernetes:**

| Network | Purpose | Example |
| --- | --- | --- |
| **Pod Network** | Pod-to-Pod communication | 10.244.0.0/16 |
| **Service Network** | Stable virtual IPs for Services | 10.96.0.0/12 |
| **Node Network** | Physical/VM network between nodes | 192.168.1.0/24 |

A **CNI (Container Network Interface)** plugin implements these rules. Popular options: Calico, Flannel, Cilium, Weave Net.

---

## Control Plane vs Worker Node — Summary Table

| Aspect | Control Plane | Worker Node |
| --- | --- | --- |
| **Role** | Makes decisions | Executes work |
| **Components** | API Server, Scheduler, Controller Manager, etcd | Kubelet, Kube Proxy, Container Runtime |
| **Runs user apps?** | No (by default, via taint) | Yes |
| **Can there be multiple?** | Yes (HA setup, 3 or 5) | Yes (scale as needed) |
| **What if it fails?** | Can't make changes, but existing Pods keep running | Pods on that node are lost, rescheduled elsewhere |
| **Network access** | Should be restricted | Needs access to API Server |

---

## Common Interview Questions

### Q: What is the role of the API Server in Kubernetes?
The API Server is the central management hub. It is the **only** component that communicates with etcd. All other components (Scheduler, Controllers, Kubelet) interact with the cluster through the API Server. It handles authentication, authorization, validation, and serves as the RESTful frontend for the cluster.

### Q: What happens if etcd goes down?
Existing workloads continue running because kubelet independently maintains containers. However, no changes can be made — no new deployments, scaling, or updates. The cluster is effectively frozen in its current state.

### Q: What is the difference between Scheduler and Controller Manager?
- **Scheduler**: Decides *where* to place new Pods (node selection)
- **Controller Manager**: Decides *what* actions to take to match desired state (create Pods, restart, scale)

### Q: How does Kubernetes ensure high availability?
- **Control plane**: Run multiple instances of API Server, Scheduler, Controller Manager. etcd runs in a cluster of 3 or 5 nodes.
- **Worker nodes**: ReplicaSets ensure Pods are spread across nodes. If a node dies, Pods are rescheduled to healthy nodes.
- **Pod Disruption Budgets**: Ensure a minimum number of Pods remain available during voluntary disruptions.

### Q: What is the difference between kubelet and kube-proxy?
- **Kubelet**: Manages Pod lifecycle — starts, stops, monitors, and reports on containers.
- **Kube-proxy**: Manages networking — ensures traffic sent to a Service IP reaches the correct Pod.

### Q: Can a node be both a control plane and a worker?
Yes, especially in small/dev clusters (like minikube or k3s). In production, control plane nodes are typically **tainted** to prevent user workloads from running on them.

### Q: What is the reconciliation loop?
The fundamental pattern in Kubernetes: controllers continuously compare the **desired state** (what the user declared) with the **actual state** (what's running). If they differ, the controller takes action to reconcile. This is what makes Kubernetes **self-healing**.

### Q: How does Kubernetes differ from Docker Swarm?
| Feature | Kubernetes | Docker Swarm |
| --- | --- | --- |
| Complexity | Higher, steep learning curve | Simpler to set up |
| Scaling | Auto-scaling built-in | Manual scaling |
| Load balancing | Advanced (Ingress, Services) | Basic |
| Community | Massive, industry standard | Smaller |
| Self-healing | Robust | Basic |
| Rolling updates | Sophisticated (canary, blue-green) | Basic |

---

## Architecture Diagram
![Cluster Architecture](/DevOps/Kubernetes/images/Kubernetes/kubernetes-cluster-architecture.svg)
![Cluster Architecture 2](/DevOps/Kubernetes/images/Kubernetes/kubernetes_architecture.gif)


## Key `kubectl` Commands for Architecture Exploration

```bash
# View cluster info
kubectl cluster-info

# List all nodes and their status
kubectl get nodes -o wide

# Describe a node (see capacity, allocatable resources, conditions)
kubectl describe node <node-name>

# View all control plane Pods (in kube-system namespace)
kubectl get pods -n kube-system

# Check component health
kubectl get componentstatuses   # deprecated but still works in some versions
kubectl get --raw='/healthz'    # API Server health

# View all API resources the cluster supports
kubectl api-resources

# View etcd entries (from inside the etcd Pod)
etcdctl get / --prefix --keys-only
```

---

## References
1. [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
1. [Kubernetes Architecture](https://saimanasak.medium.com/understanding-kubernetes-architecture-add159d720fd)
1. [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
1. [Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
1. [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/)
