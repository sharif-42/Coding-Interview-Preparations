# Kubernetes Architecture
A Kubernetes cluster consists of a control plane plus a set of worker machines, called nodes, that run containerized applications. Every cluster needs at least one worker node in order to run Pods.

The worker node(s) host the Pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster. In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.

Kubernetes has two main parts:
1. Control Plane (Brain)
2. Worker Nodes (Workers)

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
This is the front door of Kubernetes. Everything talks through API Server.

Real life analogy: `Head office receptionist`. You say:

`Run 3 copies of my Django app`

Request goes to API Server first.

Technical name: `kube-apiserver`

### Scheduler (The Decision Maker)
Scheduler decides: Which server should run your app?

Example: You have 3 servers:
- Server A → busy
- Server B → free
- Server C → medium load

Scheduler picks Server B.

Real life analogy: `Manager assigning delivery worker to location.`

Technical name: `kube-scheduler`

### Controller Manager (The Supervisor)
This watches everything and fixes problems.
You want: `3 containers running` but `Only 2 running`

Controller detects problem and starts 1 more automatically.

Real life analogy: `Supervisor ensuring enough workers are present.`

Technical name: `kube-controller-manager`

### etcd (The Database)

Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data. This stores ALL Kubernetes information. Stores:
- Which apps are running
- How many containers
- Configuration
- Cluster state

Real life analogy: `Company records database.`

Technical name: `etcd`

## Worker Nodes (The Workers)

Worker nodes actually run your applications. Each worker node has 3 components:

### Kubelet (Node Manager)

An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.Kubelet talks to control plane.

Real life analogy: Delivery center manager.

Technical name: `kubelet`

### Container Runtime (Runs containers)

A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment. This actually runs containers. Example:
- Docker
- containerd

This runs your Django app container.

Real life analogy: Kitchen where food is cooked.

### Kube Proxy (Network Manager)

kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept. Handles network communication. Ensures traffic reaches correct container.

Real life analogy: `Traffic controller directing delivery routes.`

Technical name: `kube-proxy`


## Architecture Diagram
![Cluster Architecture](/DevOps/Kubernetes/images/Kubernetes/kubernetes-cluster-architecture.svg)
![Cluster Architecture 2](/DevOps/Kubernetes/images/Kubernetes/kubernetes_architecture.gif)


## References
1. [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
1. [Kubernetes Architecture](https://saimanasak.medium.com/understanding-kubernetes-architecture-add159d720fd)
