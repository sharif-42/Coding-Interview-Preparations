# Kubernetes Architecture

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

## etcd (The Database)

This stores ALL Kubernetes information. Stores:
- Which apps are running
- How many containers
- Configuration
- Cluster state

Real life analogy: `Company records database.`

Technical name: `etcd`