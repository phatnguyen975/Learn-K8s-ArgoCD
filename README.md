<div align="center">
  <h1>K8s and ArgoCD Tutorial</h1>
  <small>
    <strong>Author:</strong> Nguyễn Tấn Phát
  </small> <br />
  <sub>February 19, 2026</sub>
</div>

## Table of Contents

- [1. Introduction to Kubernetes](#1-introduction-to-kubernetes)
- [2. Kubernetes Architecture](#2-kubernetes-architecture)
- [3. Environment Setup](#3-environment-setup)
- [4. Core Concepts (The Foundation)](#4-core-concepts-the-foundation)
- [5. Workload Management (Controllers)](#5-workload-management-controllers)
- [6. Networking](#6-networking)

## 1. Introduction to Kubernetes

### What is Kubernetes?

**Kubernetes (K8s)** is an open-source **container orchestration system** used to:

- Deploy containerized applications
- Scale them automatically
- Manage networking
- Handle self-healing
- Perform rolling updates

Originally developed by Google and now maintained by Cloud Native Computing Foundation (CNCF).

### Why Kubernetes?

**Before Kubernetes:**

- We used containers (e.g., Docker)
- But managing many containers across multiple servers was hard

**Kubernetes solves:**

- High availability
- Auto-scaling
- Rolling deployment
- Service discovery
- Load balancing
- Secret management

## 2. Kubernetes Architecture

### Cluster Architecture

A Kubernetes cluster consists of two main types of resources:

**1. Control Plane (Master Node):** The brain of the cluster.

- **API Server:** The entry point for all REST commands (`kubectl` talks to this).
- **Etcd:** A key-value store backing the cluster data (the "database" of K8s).
- **Scheduler:** Assigns newly created pods to nodes.
- **Controller Manager:** Handles background processes (e.g., detecting if a node goes down).

**2. Worker Nodes:** The machines (VMs or physical servers) where applications run.

- **Kubelet:** An agent that ensures containers are running in a Pod.
- **Kube-proxy:** Maintains network rules on nodes.
- **Container Runtime:** Software that runs containers (e.g., Docker, containerd).

### Kubernetes Object Model

Everything in K8s is an object defined in YAML.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

|    Field     |    Meaning    |
| :----------: | :-----------: |
| `apiVersion` |   API group   |
|    `kind`    |  Object type  |
|  `metadata`  |  Object info  |
|    `spec`    | Desired state |

## 3. Environment Setup

You have multiple options:

- **Local single-node:** [Minikube](https://minikube.sigs.k8s.io/docs/)
- **CLI-only lightweight:** [K3s](https://docs.k3s.io/)
- **Production:** kubeadm
- **Cloud:** EKS / GKE / AKS

To follow this tutorial, we will use **Minikube** (a local single-node cluster) and **kubectl** (the CLI tool).

**Step 1: Install `kubectl` (The CLI)**

- **Description:** This tool allows you to run commands against Kubernetes clusters.
- **OS:** Linux (compatible with WSL2).

```bash
# 1. Download the latest release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 2. Validate the binary (optional but recommended)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# 3. Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 4. Verify installation
kubectl version --client
```

**Step 2: Install Minikube (Local Cluster)**

- **Description:** Runs a single-node K8s cluster inside a VM or Docker container.

```bash
# 1. Download and install
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# 2. Start the cluster (Using Docker as driver is recommended)
minikube start --driver=docker

# 3. Check cluster info and status
kubectl cluster-info
kubectl get nodes
```

## 4. Core Concepts (The Foundation)

### Pods

**Definition:** The smallest deployable unit in K8s. A Pod represents a single instance of a running process (usually one container, sometimes more).

**YAML Structure (`pod.yaml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: my-website
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

**Key Commands:**

|                Command                 |                                Explanation                                |                  Example                   |
| :------------------------------------: | :-----------------------------------------------------------------------: | :----------------------------------------: |
|       `kubectl apply -f <file>`        |           Creates or updates resources defined in a YAML file.            |        `kubectl apply -f pod.yaml`         |
|           `kubectl get pods`           |                 Lists all pods in the current namespace.                  | `kubectl get pods -o wide` (shows IP/Node) |
|     `kubectl describe pod <name>`      | Shows detailed info (events, errors, config). **Critical for debugging.** |      `kubectl describe pod nginx-pod`      |
|         `kubectl logs <name>`          |                Prints the stdout/stderr of the container.                 |          `kubectl logs nginx-pod`          |
| `kubectl exec -it <name> -- /bin/bash` |             Opens an interactive shell inside the container.              | `kubectl exec -it nginx-pod -- /bin/bash`  |

### Namespaces

**Definition:** A mechanism to isolate groups of resources within a single cluster (e.g., `dev`, `staging`, `prod`).

```bash
# Create a namespace
kubectl create namespace dev

# Run a pod inside that namespace
kubectl apply -f pod.yaml -n dev
```

## 5. Workload Management (Controllers)

Managing individual Pods is manual and risky. We use Controllers to manage them.

### Deployment

**Definition:** Manages a set of identical Pods (via ReplicaSets). It ensures the desired number of pods are running and handles rolling updates.

**YAML Structure (`deployment.yaml`):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3 # Desired number of pods
  selector:
    matchLabels:
      app: my-app # Must match the template labels below
  template: # The blueprint for the Pods
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

**Key Commands:**

|                            Command                            |                            Explanation                            |
| :-----------------------------------------------------------: | :---------------------------------------------------------------: |
|                   `kubectl get deployments`                   |                         List deployments.                         |
|              `kubectl apply -f deployment.yaml`               | Apply the desired state defined in the YAML file to your cluster. |
|        `kubectl scale deployment <name> --replicas=5`         |           Manually scale the number of pods up or down.           |
| `kubectl set image deployment <name> <container>=<new-image>` |       Update the image version (triggers a Rolling Update).       |
|          `kubectl rollout status deployment <name>`           |                 Watch the progress of an update.                  |
|           `kubectl rollout undo deployment <name>`            |     Rollback to the previous version if the new one crashes.      |

## 6. Networking

Pods are ephemeral (they die and change IPs). **Services** provide a stable network endpoint.

### Service Types

1. **ClusterIP (Default):** Exposes the Service on an internal IP. Only accessible within the cluster.
2. **NodePort:** Exposes the Service on each Node's IP at a static port (30000-32767). Accessible externally.
3. **LoadBalancer:** Exposes the Service externally using a cloud provider's Load Balancer (AWS ELB, GCP LB).

**YAML Structure (`service.yaml`):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort # or ClusterIP, LoadBalancer
  selector:
    app: my-app # Connects this service to pods with label 'app: my-app'
  ports:
    - protocol: TCP
      port: 80 # Port exposed by the Service
      targetPort: 80 # Port running inside the Container
      nodePort: 30007 # (Optional) Specific port for external access
```

## References

- [Kubernetes](https://www.youtube.com/watch?v=X48VuDVv0do)
- [ArgoCD](https://www.youtube.com/watch?v=MeU5_k9ssrs)
