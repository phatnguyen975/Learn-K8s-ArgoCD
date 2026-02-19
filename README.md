<div align="center">
  <h1>K8s and ArgoCD Tutorial</h1>
  <small>
    <strong>Author:</strong> Nguyễn Tấn Phát
  </small> <br />
  <sub>February 19, 2026</sub>
</div>

## Table of Contents

1. [Introduction to Kubernetes](#1-introduction-to-kubernetes)
2. [Kubernetes Architecture](#2-kubernetes-architecture)
3. [Environment Setup](#3-environment-setup)
4. [Core Concepts (The Foundation)](#4-core-concepts-the-foundation)
5. [Workload Management (Controllers)](#5-workload-management-controllers)
6. [Networking](#6-networking)
7. [Configuration & Storage](#7-configuration--storage)
8. [Advanced Concepts](#8-advanced-concepts)

## 1. Introduction to Kubernetes

### What is Kubernetes?

**Kubernetes** (often abbreviated as **K8s**) is a powerful, open-source **container orchestration platform**. Originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF).

It is designed to automate:

- Deployment of containerized applications
- Scaling applications
- Managing container networking
- Self-healing
- Rolling updates and rollbacks

### Understanding "Container Orchestration"

To understand Kubernetes, you first need to understand the problem it solves. Packaging an application (like a Java Spring Boot backend) into a Docker container makes it easy to run on a single machine. However, when you move to a production environment, managing these containers becomes incredibly complex.

If you have an application that requires 50 running containers across 10 different servers, you face several challenges:

- How do you ensure they communicate securely?
- What happens if one of the servers crashes?
- How do you update the application to a new version without taking the system offline?

Kubernetes acts as the **"orchestrator"** or the brain that manages this entire distributed system. You declare the desired state of your system (e.g., "I want 3 instances of my web application running at all times"), and Kubernetes constantly monitors the environment to ensure the actual state matches your desired state.

Therefore, container orchestration means:

- Scheduling containers on nodes
- Restarting them if they crash
- Replacing them if node fails
- Scaling them up/down
- Managing communication between them

Kubernetes does orchestration at cluster scale.

### Key Benefits of Kubernetes

Kubernetes is the industry standard for cloud-native applications because it provides a robust set of features out of the box.

- **Automated Rollouts and Rollbacks**
  Kubernetes allows you to describe the desired state for your deployed containers. When integrated with CI/CD tools like Jenkins, you can automate the process of rolling out new code. If you deploy a new version of your application, Kubernetes will progressively update the containers one by one to ensure zero downtime. If the new version fails or has a bug, Kubernetes can automatically roll back to the previous, stable version.

- **Self-Healing Capabilities**
  Kubernetes constantly monitors the health of your nodes and containers. If a container crashes, Kubernetes automatically restarts it. If a worker node fails entirely, Kubernetes will reschedule the containers that were running on that node to other healthy nodes in the cluster. It also kills containers that do not respond to your user-defined health checks and does not route traffic to them until they are ready to serve requests.

- **Service Discovery and Load Balancing**
  Containers are ephemeral; they are created and destroyed frequently, meaning their IP addresses constantly change. Kubernetes solves this by giving containers their own IP addresses and a single DNS name for a set of containers. It can also distribute network traffic (load balancing) across these containers so that the deployment is stable, without requiring you to modify your application code.

- **Horizontal Scaling**
  You can scale your application up and down quickly and easily. This can be done manually via a simple command or a UI, or automatically based on CPU usage, memory consumption, or even custom metrics. During a traffic spike, Kubernetes can rapidly spin up new container instances to handle the load and then terminate them when the traffic subsides to save resources.

- **Secret and Configuration Management**
  Kubernetes lets you store and manage sensitive information, such as OAuth tokens, database passwords, and SSH keys. You can deploy and update these secrets and application configuration separately from your container images. This means you do not need to rebuild your Docker images or expose secrets in your stack configuration just to change a database password.

- **Storage Orchestration**
  Kubernetes allows you to automatically mount the storage system of your choice. Whether your containers need local storage, storage from a public cloud provider (like AWS or GCP), or a network storage system (like NFS), Kubernetes manages the lifecycle of that storage seamlessly.

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

## 7. Configuration & Storage

Separating configuration from container images is a best practice (The 12-Factor App).

### ConfigMaps & Secrets

- **ConfigMap:** Stores non-sensitive data (config files, environment variables).
- **Secret:** Stores sensitive data (passwords, keys) encoded in Base64.

**Example: Creating a Secret (CLI method)**

```bash
# Create a secret named "db-pass"
kubectl create secret generic db-pass --from-literal=password="MySuperSecretPass"

# Verify
kubectl get secrets
```

### Persistent Volumes (PV) & Persistent Volume Claims (PVC)

Containers are ephemeral; when they restart, file system data is lost. PV/PVC provides permanent storage.

- **PV:** A piece of storage in the cluster (Physical Hard Drive, AWS EBS).
- **PVC:** A request for storage by a user (e.g., "I need 5GB").

**Flow:** Pod requests -> PVC -> binds to -> PV.

## 8. Advanced Concepts

### Ingress

**Definition:** An API object that manages external access to the services in a cluster, typically HTTP/HTTPS. It acts as a smart router/reverse proxy (like Nginx) sitting in front of your Services.

**Why use it?** Instead of using 10 LoadBalancers for 10 Services (expensive), you use 1 Ingress Controller to route traffic based on domains (e.g., `api.example.com` -> Service A, `web.example.com` -> Service B).

**Setup (Minikube):**

```bash
# Enable NGINX Ingress controller in Minikube
minikube addons enable ingress
```

**YAML Structure (`ingress.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

### Helm (The Package Manager)

**Concept:** Helm is like `apt` or `yum` or `maven` but for Kubernetes. It helps you install complex applications (like Prometheus, Jenkins, MySQL) with a single command.

**Install Helm:**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Common Commands:**

```bash
# Add a repository (e.g., Bitnami)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Search for a chart
helm search repo mysql

# Install a chart
helm install my-db bitnami/mysql

# List releases
helm list
```

## References

- [Kubernetes in a Nutshell](https://medium.com/swlh/kubernetes-in-a-nutshell-tutorial-for-beginners-caa442dfd6c0)
- [Kubernetes Course](https://www.youtube.com/watch?v=X48VuDVv0do)
- [ArgoCD Course](https://www.youtube.com/watch?v=MeU5_k9ssrs)
