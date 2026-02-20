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


## Simple definition
`Kubernetes is a system that automatically runs and manages your containers. It makes sure.`

- The apps are running
- Correct number of copies exist
- Crashed apps restart automatically
- Traffic is distributed properly
- Scaling happens automatically

Kubernetes is often abbreviated as K8s.


### Real-life analogy: Restaurant manager

Think of containers as cooks. Kubernetes is the restaurant manager. Manager’s job:

- Make sure enough cooks are working
- Replace sick cooks immediately
- Assign orders evenly
- Add more cooks during rush hour
- Remove extra cooks when quiet

You don’t manage cooks manually. Manager handles everything. `Kubernetes is that manager`.


## Core building blocks (simple explanation)

### 1. Container

Your app packaged with everything needed. Example: Django app inside Docker container

### 2. Pod

Smallest unit in Kubernetes.

Pod = container runner

Think: Pod → runs your container

### 3. Node
A server (machine). Example:

- DigitalOcean server
- AWS server
- Your VM

Node runs Pods.

### 4. Cluster
Group of nodes.

Cluster = multiple servers working together

Kubernetes manages the cluster.

### 5. Deployment

Defines:

- Which app to run
- How many copies
- Which container image

Example: Run 3 copies of Django app.

### 6. Service

Provides stable access to your app.

Example: Users access your app via one IP, even if containers change.