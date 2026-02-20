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
4. [The Developer's Toolkit: Mastering `kubectl`](#4-the-developers-toolkit-mastering-kubectl)
5. [Anatomy of a Kubernetes YAML Manifest](#5-anatomy-of-a-kubernetes-yaml-manifest)
6. [Core Components](#6-core-components) \
   6.1. [Pod](#pod) \
   6.2. [ReplicaSet](#replicaset) \
   6.3. [Deployment](#deployment) \
   6.4. [Service](#service) \
   6.5. [Namespace](#namespace)
7. [Configuration Management: ConfigMaps and Secrets](#7-configuration-management-configmaps-and-secrets)
8. [Storage: Persistent Volumes (PV) & Persistent Volume Claims (PVC)](#8-storage-persistent-volumes-pv--persistent-volume-claims-pvc)
9. [Ingress: The Smart Router (Layer 7 Load Balancing)](#9-ingress-the-smart-router-layer-7-load-balancing)
10. [Helm (The Package Manager)](#10-helm-the-package-manager)
11. [StatefulSets: Managing Stateful Applications](#11-statefulsets-managing-stateful-applications)
12. [Security: Role-Based Access Control (RBAC)](#12-security-role-based-access-control-rbac)
13. [ArgoCD & GitOps: The Modern Deployment Standard](#13-argocd--gitops-the-modern-deployment-standard)

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

- **Automated Rollouts and Rollbacks** \
  Kubernetes allows you to describe the desired state for your deployed containers. When integrated with CI/CD tools like Jenkins, you can automate the process of rolling out new code. If you deploy a new version of your application, Kubernetes will progressively update the containers one by one to ensure zero downtime. If the new version fails or has a bug, Kubernetes can automatically roll back to the previous, stable version.

- **Self-Healing Capabilities** \
  Kubernetes constantly monitors the health of your nodes and containers. If a container crashes, Kubernetes automatically restarts it. If a worker node fails entirely, Kubernetes will reschedule the containers that were running on that node to other healthy nodes in the cluster. It also kills containers that do not respond to your user-defined health checks and does not route traffic to them until they are ready to serve requests.

- **Service Discovery and Load Balancing** \
  Containers are ephemeral; they are created and destroyed frequently, meaning their IP addresses constantly change. Kubernetes solves this by giving containers their own IP addresses and a single DNS name for a set of containers. It can also distribute network traffic (load balancing) across these containers so that the deployment is stable, without requiring you to modify your application code.

- **Horizontal Scaling** \
  You can scale your application up and down quickly and easily. This can be done manually via a simple command or a UI, or automatically based on CPU usage, memory consumption, or even custom metrics. During a traffic spike, Kubernetes can rapidly spin up new container instances to handle the load and then terminate them when the traffic subsides to save resources.

- **Secret and Configuration Management** \
  Kubernetes lets you store and manage sensitive information, such as OAuth tokens, database passwords, and SSH keys. You can deploy and update these secrets and application configuration separately from your container images. This means you do not need to rebuild your Docker images or expose secrets in your stack configuration just to change a database password.

- **Storage Orchestration** \
  Kubernetes allows you to automatically mount the storage system of your choice. Whether your containers need local storage, storage from a public cloud provider (like AWS or GCP), or a network storage system (like NFS), Kubernetes manages the lifecycle of that storage seamlessly.

## 2. Kubernetes Architecture

Kubernetes follows a classic distributed system architecture, which is split into two main parts:

- **Control Plane** (the master/brain that manages the cluster)
- **Worker Nodes** (the muscle where your actual applications run)

<p align="center">
  <img src="https://www.simform.com/wp-content/uploads/2023/08/Kubernetes-Architecture-Diagram.jpg" style="width:90%;" alt="K8s Architecture">
</p>

### The Control Plane (Master Node)

The **Control Plane** is responsible for maintaining the desired state of the cluster. It makes global decisions (like scheduling) and detects/responds to cluster events (like starting up a new container when one crashes). For high availability in production, the Control Plane components are usually replicated across multiple machines.

Here are the core components of the Control Plane:

- **API Server (`kube-apiserver`)**
  - **What it is:** The front-end of the Kubernetes control plane. It exposes the Kubernetes API.
  - **How it works:** Whenever you type a command using `kubectl` (or when a CI/CD pipeline interacts with the cluster), that request goes directly to the API Server. It validates the request, processes it, and updates the state of the cluster. It is the only component that communicates directly with the cluster's datastore (`etcd`).
  - **Analogy:** Think of it as the central REST API gateway for the entire system. All other components, both internal and external, must talk to the API Server to get or set information.
- **Etcd (`etcd`)**
  - **What it is:** A consistent, highly available, distributed key-value store.
  - **How it works:** This is the "database" of your Kubernetes cluster. It stores all the cluster data, including the current state, configuration details, and secrets. Because it is critical to the cluster's survival, it must be backed up regularly. If `etcd` goes down or loses data, the cluster loses its brain and forgets what applications are supposed to be running.
- **Scheduler (`kube-scheduler`)**
  - **What it is:** The component responsible for assigning work to the Worker Nodes.
  - **How it works:** When you create a new deployment, the API Server registers that new Pods need to be created, but it doesn't know where to put them. The Scheduler constantly watches the API Server for newly created Pods that have no assigned Node. It then evaluates the resource requirements (CPU, RAM) of the Pod, checks the available capacity on all Worker Nodes, and assigns the Pod to the best-fitting Node.
- **Controller Manager (`kube-controller-manager`)**
  - **What it is:** A daemon that runs continuous control loops to regulate the state of the cluster.
  - **How it works:** It constantly compares the actual state of the cluster against the desired state stored in `etcd`. It includes several specific controllers:
    - **Node Controller:** Notices and responds when a worker node goes down.
    - **ReplicaSet Controller:** Ensures that the correct number of Pods are running for a given deployment.
    - **Endpoint Controller:** Populates Service network routes (joining Services and Pods).

### The Worker Nodes (Data Plane)

**Worker Nodes** are the machines (VMs or physical servers) that execute the workloads. While the Control Plane manages the cluster, the Worker Nodes actually run your application containers (like a Java application, a database, or an Nginx server).

Every Worker Node runs the following components:

- **Kubelet (`kubelet`)**
  - **What it is:** The primary "agent" that runs on every single Worker Node.
  - **How it works:** It communicates directly with the Master Node's API Server. The Kubelet receives instructions (Pod specifications) from the control plane and ensures that the containers described in those specs are actually running and healthy on its specific machine. If a container fails, the Kubelet tries to restart it.
  - **Analogy:** It acts much like an agent in an automated pipeline system, receiving instructions from a master server and executing the local heavy lifting.
- **Kube Proxy (`kube-proxy`)**
  - **What it is:** A network proxy that runs on each node in your cluster.
  - **How it works:** It maintains network rules on the host machine using operating system packet filtering (like `iptables` in Linux). These rules allow network communication to your Pods from network sessions inside or outside of your cluster. When a Service is created, `kube-proxy` ensures that traffic destined for that Service's IP is properly routed to the correct backend Pod.
- **Container Runtime**
  - **What it is:** The underlying software that is responsible for actually running the containers.
  - **How it works:** Kubernetes doesn't run containers itself; it tells the Container Runtime to do it. While Docker used to be the default, Kubernetes now supports any runtime that implements the Kubernetes Container Runtime Interface (CRI), such as `containerd` or `CRI-O`.

## 3. Environment Setup

You have multiple options:

- **Local single-node:** [Minikube](https://minikube.sigs.k8s.io/docs/)
- **CLI-only lightweight:** [K3s](https://docs.k3s.io/)
- **Production:** kubeadm
- **Cloud:** EKS / GKE / AKS

To follow this tutorial, we will use **Minikube** (a local single-node cluster) and **kubectl** (the CLI tool).

- **kubectl:** The Kubernetes command-line tool. This is your steering wheel. You use it to run commands against Kubernetes clusters to deploy applications, inspect and manage cluster resources, and view logs.
- **Minikube:** A lightweight Kubernetes implementation that creates a VM or container on your local machine and deploys a simple, single-node cluster inside it.

Before starting, ensure you have a container runtime installed, such as **Docker**, as Minikube will need it to provision the cluster nodes.

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

# 3. Verify kubectl is connected to Minikube
kubectl cluster-info
kubectl get nodes
```

- **Useful Commands:**
  - `minikube status`: Checks the health of the Minikube components (host, kubelet, apiserver).
  - `minikube ip`: Returns the IP address of the local cluster (useful for testing networking later).
  - `minikube dashboard`: Kubernetes comes with a built-in web UI. This command enables the dashboard add-on and opens it in your default web browser, giving you a visual representation of your cluster.
  - `minikube stop`: Safely shuts down the local cluster without deleting your deployed resources.
  - `minikube delete`: Completely destroys the cluster and wipes all data.

## 4. The Developer's Toolkit: Mastering `kubectl`

[Kubernetes Cheatsheet](https://phoenixnap.com/kb/wp-content/uploads/2021/11/kubectl-commands-cheat-sheet-by-pnap.pdf)

`kubectl` is your primary interface with the Kubernetes Control Plane. It operates in two main ways:

1. **Imperative:** Telling K8s exactly what to do step-by-step (e.g., "Create a pod named X with image Y"). Good for quick tests.
2. **Declarative:** Telling K8s what you want the end result to be using YAML files (e.g., "Make the cluster look like this file"). This is the industry standard for production.

Below is a comprehensive guide to the commands you will use daily, categorized by their function.

### Command Structure

**General Syntax:**

```bash
kubectl <verb> <resource> <name> <flags>
```

**Resource Types:**

|        Short         |         Full          |
| :------------------: | :-------------------: |
|         `po`         |          pod          |
|         `no`         |         node          |
|       `deploy`       |      deployment       |
|        `svc`         |        service        |
|         `rs`         |      replicaset       |
|         `ds`         |       daemonset       |
|         `ns`         |       namespace       |
|         `cm`         |       configmap       |
|         `ep`         |       endpoints       |
|       `secret`       |        secret         |
|        `pvc`         | persistentvolumeclaim |
|         `pv`         |   persistentvolume    |
|        `ing`         |        ingress        |
|        `sts`         |      statefulset      |
|         `cj`         |        cronjob        |
|        `job`         |          job          |
|         `sa`         |    serviceaccount     |
|        `role`        |         role          |
|    `rolebinding`     |      rolebinding      |
|    `clusterrole`     |      clusterrole      |
| `clusterrolebinding` |  clusterrolebinding   |

List all supported resources:

```bash
kubectl api-resources
```

**Core `kubectl` Verbs (Commands):**

|      Verb      |                                                Purpose                                                |
| :------------: | :---------------------------------------------------------------------------------------------------: |
|     `get`      |       Retrieve a list or basic information about resources (pods, services, deployments, etc.)        |
|   `describe`   | Show detailed information about a resource, including status and events (commonly used for debugging) |
|    `create`    |            Create a resource imperatively (execute immediately, rarely used in production)            |
|    `apply`     |       Create or update resources declaratively from a YAML file (standard for CI/CD workflows)        |
|   `explain`    |                       Display documentation for a resource and its YAML fields                        |
|    `delete`    |                                  Remove a resource from the cluster                                   |
|     `edit`     |                           Edit a live resource directly using a text editor                           |
|    `scale`     |                      Increase or decrease the number of replicas for a workload                       |
|     `set`      |   Update specific parts of a resource configuration (image, environment variables, resources, etc.)   |
|   `rollout`    |                   Manage deployment rollouts (check status, view history, rollback)                   |
|     `logs`     |                              View logs from a container running in a pod                              |
|     `exec`     |                             Execute a command inside a running container                              |
|    `config`    |                        Manage kubeconfig settings (clusters, contexts, users)                         |
|     `auth`     |                               Check authorization and RBAC permissions                                |
| `port-forward` |             Forward a pod or service port to your local machine for testing and debugging             |

**Global Flags:**

|           Flag           |                                   Meaning                                    |
| :----------------------: | :--------------------------------------------------------------------------: |
|   `-n`, `--namespace`    |                 Specify the namespace to run the command in                  |
| `-A`, `--all-namespaces` |                  Run the command across **all namespaces**                   |
|     `-o`, `--output`     |         Control output format (e.g. `yaml`, `json`, `wide`, `name`)          |
|    `--dry-run=client`    | Simulate the command locally without actually creating or changing resources |
|     `--watch`, `-w`      |            Continuously watch resources for changes in real time             |
|       `--context`        |              Use a specific Kubernetes context from kubeconfig               |
|    `--field-selector`    |      Filter resources by specific fields (e.g. `status.phase=Running`)       |
|    `-l`, `--selector`    |                    Filter resources using label selectors                    |
|       `--sort-by`        |                       Sort output by a specific field                        |
|     `--show-labels`      |                         Display labels in the output                         |
|         `--all`          |    Apply the command to **all resources of a given type** in a namespace     |
|        `--force`         |          Force resource deletion (can cause immediate termination)           |
|          `--as`          |               Impersonate another user (used for RBAC testing)               |
|       `--as-group`       |                   Impersonate a user group (RBAC testing)                    |

### Cluster Management & Context

|               Command               |                                  Meaning                                   |                            When to Use                             |                Example                |
| :---------------------------------: | :------------------------------------------------------------------------: | :----------------------------------------------------------------: | :-----------------------------------: |
|       `kubectl cluster-info`        |       Displays the addresses of the control plane and core services        |       To verify cluster connectivity and API server address        |        `kubectl cluster-info`         |
|    `kubectl config get-contexts`    | Lists all available cluster contexts (environments) in your `.kube/config` | When managing multiple clusters (e.g. Minikube, AWS Prod, GCP Dev) |     `kubectl config get-contexts`     |
| `kubectl config use-context <name>` |             Switches the active context to a different cluster             |     When switching from Dev to Prod (or between environments)      | `kubectl config use-context minikube` |

### Creating & Applying Resources

|               Command                |                                     Meaning                                      |                         When to Use                          |                   Example                   |
| :----------------------------------: | :------------------------------------------------------------------------------: | :----------------------------------------------------------: | :-----------------------------------------: |
|    `kubectl apply -f <file.yaml>`    |      **(Declarative)** Creates or updates resources defined in a YAML file       | Primary deployment command for managing Kubernetes resources |     `kubectl apply -f deployment.yaml`      |
|   `kubectl apply -f <directory>/`    |                 Applies all YAML files in a specified directory                  |          Deploy an entire application stack at once          |   `kubectl apply -f ./my-app-manifests/`    |
| `kubectl run <name> --image=<image>` |          **(Imperative)** Creates a single Pod running a specific image          |      Quick debugging, image testing, or temporary tasks      | `kubectl run test-pod --image=nginx:alpine` |
|  `kubectl create <resource> <name>`  | **(Imperative)** Creates a specific resource (e.g. namespace, secret, configmap) | Quickly generate simple resources without writing full YAML  |     `kubectl create namespace prod-env`     |

### Viewing & Finding Resources (`get`)

The `get` command lists resources. You can append `-n <namespace>` to any of these to look outside the default namespace, or `-A` for all namespaces.

|             Command              |                                   Meaning                                    |                                                      When to Use                                                       |               Example               |
| :------------------------------: | :--------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------: | :---------------------------------: |
|        `kubectl get pods`        |                   Lists all Pods in the current namespace                    | Check whether applications are running, starting, or failing (e.g. `Running`, `ContainerCreating`, `CrashLoopBackOff`) |         `kubectl get pods`          |
| `kubectl get <resource> -o wide` | Lists resources with additional details (node assignment, internal IP, etc.) |                             See where a pod is running and its network-related information                             |     `kubectl get pods -o wide`      |
| `kubectl get <resource> -o yaml` |                Outputs the full YAML definition of a resource                |        Back up a resource or inspect how Kubernetes modified your configuration (default values, status fields)        | `kubectl get deploy my-app -o yaml` |
|        `kubectl get all`         | Lists common resources such as Pods, Services, Deployments, and ReplicaSets  |                               Get a quick overview of everything running in a namespace                                |      `kubectl get all -n dev`       |

### Inspecting & Debugging (`describe`, `logs`, `exec`)

These are the most critical commands for troubleshooting when things go wrong.

|                Command                 |                                         Meaning                                          |                                                                 When to Use                                                                 |                    Example                     |
| :------------------------------------: | :--------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------------------------------------------------------: | :--------------------------------------------: |
|  `kubectl describe <resource> <name>`  | Shows detailed, human-readable information about a resource, including recent **Events** | When a Pod fails to start or behaves unexpectedly; the **Events** section explains the reason (e.g. `ImagePullBackOff`, `Insufficient CPU`) |      `kubectl describe pod my-nginx-pod`       |
|       `kubectl logs <pod-name>`        |            Prints the standard output (stdout/stderr) of a container in a Pod            |                                           Read application logs, stack traces, or crash messages                                            |         `kubectl logs my-backend-pod`          |
|      `kubectl logs -f <pod-name>`      |                     Streams logs in real time (similar to `tail -f`)                     |                                               Monitor live traffic or debug a running process                                               |        `kubectl logs -f my-backend-pod`        |
| `kubectl exec -it <pod-name> -- <cmd>` |      Executes a command inside a running container; `-it` enables interactive mode       |                  Open a shell to inspect files, test network connectivity (`curl`, `ping`), or check environment variables                  | `kubectl exec -it db-pod -- /bin/bash` or `sh` |

### Modifying & Deleting

|                      Command                       |                                 Meaning                                  |                                             When to Use                                              |                   Example                   |
| :------------------------------------------------: | :----------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------: | :-----------------------------------------: |
|          `kubectl edit <resource> <name>`          | Opens the live resource in your default terminal editor (e.g. Vim, Nano) |       Quick, temporary fixes or tests _(changes may be overwritten by future `kubectl apply`)_       |        `kubectl edit service my-svc`        |
| `kubectl scale deployment <name> --replicas=<num>` |     Immediately changes the number of running Pods for a Deployment      |                         Handle traffic spikes or scale down to reduce costs                          | `kubectl scale deploy web-app --replicas=5` |
|          `kubectl delete -f <file.yaml>`           |         Deletes all resources defined in the specified YAML file         |                 Completely remove an application stack that was previously deployed                  |     `kubectl delete -f deployment.yaml`     |
|         `kubectl delete <resource> <name>`         |                 Imperatively deletes a specific resource                 | Remove a stuck Pod (will be recreated if managed by a Deployment) or delete a resource like a Secret |       `kubectl delete pod faulty-pod`       |

## 5. Anatomy of a Kubernetes YAML Manifest

### The 4 Universal Root Fields

**1. `apiVersion`:** Which version of the Kubernetes API you're using to create this object.

- **Examples:** `v1` (for Pods, Services, ConfigMaps), `apps/v1` (for Deployments, StatefulSets), `networking.k8s.io/v1` (for Ingress).

**2. `kind`:** What kind of object you want to create.

- **Examples:** `Pod`, `Deployment`, `Service`, `Secret`, `Ingress`. (**Note:** Case-sensitive, always capitalized)

**3. `metadata`:** Data that uniquely identifies the object.

- `name`: The **unique** name of the resource **(required)**.
- `namespace`: Where it lives (defaults to `default`).
- `labels`: Key-value pairs used to organize and select resources (e.g., `app: frontend`, `env: prod`). **Crucial for linking resources together.**

**4. `spec`:** The actual specification or "meat" of the resource. This field dictates exactly what the resource should look like and how it behaves. The contents of `spec` are entirely dependent on the `kind`.

### The Pod `spec`

The Pod `spec` defines the containers running inside it.

```yaml
spec:
  # [1. Containers Array] - A Pod can have one or more containers.
  containers:
    - name: my-app-container # Name of the container inside the pod
      image: my-repo/my-app:1.0.0 # The Docker image to pull
      imagePullPolicy: IfNotPresent # When to pull (Always, IfNotPresent, Never)

      # [2. Ports] - Which ports the container is listening on
      ports:
        - containerPort: 8080 # The port your app is actually using

      # [3. Environment Variables] - Passing config to the container
      env:
        - name: DATABASE_URL # Standard key-value
          value: "jdbc:mysql://db:3306/mydb"
        - name: DB_PASSWORD # Pulling a value from a K8s Secret
          valueFrom:
            secretKeyRef:
              name: my-db-secret
              key: password

      # [4. Resource Requests & Limits] - VERY IMPORTANT for stability
      resources:
        requests: # What the pod needs to start (Scheduler uses this)
          memory: "256Mi"
          cpu: "250m" # 250 millicores (1/4 of a CPU)
        limits: # The *maximum* it can use before getting killed (OOMKilled)
          memory: "512Mi"
          cpu: "500m"
```

### The Deployment `spec`

A Deployment wraps around a Pod to provide scaling and updates.

```yaml
spec:
  # [1. Replicas] - How many identical Pods do you want running?
  replicas: 3

  # [2. Selector] - How does the Deployment know which Pods belong to it?
  selector:
    matchLabels:
      app: my-web-app # It will manage all pods with this label

  # [3. Strategy] - How to perform updates?
  strategy:
    type: RollingUpdate # Updates pods one by one (Zero downtime)
    rollingUpdate:
      maxSurge: 1 # Can create 1 extra pod during update
      maxUnavailable: 0 # Cannot drop below desired replica count during update

  # [4. Template] - The actual Pod blueprint. Notice it contains 'metadata' and 'spec' just like a Pod!
  template:
    metadata:
      labels:
        app: my-web-app # MUST match the selector above!
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### The Service `spec`

A Service provides networking and load balancing.

```yaml
spec:
  # [1. Type] - How the service is exposed
  type: ClusterIP # Default. Internal access only. (Others: NodePort, LoadBalancer)

  # [2. Selector] - Which Pods should receive this traffic?
  selector:
    app: my-web-app # Routes traffic to pods with this label

  # [3. Ports Array] - Mapping service ports to pod ports
  ports:
    - protocol: TCP # Default. (Others: UDP, SCTP)
      port: 80 # The port the Service exposes to the cluster
      targetPort: 8080 # The port the actual Container is listening on
      # nodePort: 30005 # Only used if type is NodePort (Exposes port on host machine)
```

## 6. Core Components

### Pod

**1. What is a Pod?**

In the Docker ecosystem, the smallest unit you deploy is a single container. However, Kubernetes does not run containers directly; it wraps one or more containers into a higher-level structure called a **Pod**.

A Pod is the smallest, most basic, and deployable compute unit you can create and manage in Kubernetes.

**2. The "Logical Host" Concept**

You can think of a Pod as a tiny, isolated "logical host" (similar to a lightweight Virtual Machine or a WSL2 instance). When multiple containers are placed inside the same Pod, they behave as if they are running on the same physical machine.

This means containers within the same Pod share:

- **Network Namespace:** All containers in a Pod share the same IP address and port space. If you have a Java Spring Boot application container running on port 8080 and a logging container in the same Pod, they can communicate with each other using `localhost:8080`.
- **Storage (Volumes):** Containers in the same Pod can mount the same shared storage volumes, allowing them to read and write to the exact same files simultaneously.
- **IPC (Inter-Process Communication):** They can communicate using standard Linux IPC mechanisms (like shared memory).

**3. Single-Container vs. Multi-Container Pods**

- **One-Container-per-Pod (The Standard):** This is the most common use case. You have one container (e.g., a Spring Boot API) running inside one Pod. K8s manages the Pod, and the Pod manages the container.
- **Multi-Container Pods (The Sidecar Pattern):** You put multiple containers in one Pod only if they are tightly coupled and must run together on the exact same server.
  - **Example:** Your main container is a web server, and your second container is a "Sidecar" that pulls the latest configuration files from a Git repository every 5 minutes and saves them to a shared volume that the web server reads.

**4. The Pod Lifecycle (Phases)**

Pods are ephemeral (temporary). They are not designed to live forever. When a Pod is created, it goes through several phases:

- **Pending:** The API Server accepted the Pod, but the Scheduler is still looking for a Worker Node to place it on, or the container image is currently downloading.
- **Running:** The Pod has been bound to a Node, and all containers have been created. At least one container is running.
- **Succeeded:** All containers in the Pod have terminated successfully (exit code 0) and will not be restarted. (Common for batch jobs).
- **Failed:** All containers have terminated, but at least one failed (exited with a non-zero status, like an application crash).
- **CrashLoopBackOff:** A very common state. It means a container repeatedly crashes immediately after starting, and Kubernetes is waiting longer and longer intervals before trying to restart it again.

**5. YAML Specification (`pod.yaml`)**

Here is a comprehensive YAML example of a Pod running a Java application, detailing the most important `spec` configurations.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: library-api-pod # The unique name of the Pod
  labels: # Used by Services and Deployments to find this Pod
    app: library-system
    tier: backend
spec:
  # [1. Containers Array] Defines what runs inside the Pod
  containers:
    - name: spring-boot-app # Name of the container
      image: myrepo/library-api:v1.2.0 # The Docker image to pull
      imagePullPolicy: IfNotPresent # Only pull if it's not cached locally

      # [2. Ports] Documents which ports the container uses
      ports:
        - containerPort: 8080
          protocol: TCP

      # [3. Environment Variables] Passed to the Linux OS inside the container
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_HOST
          value: "postgres-service.default.svc.cluster.local"

      # [4. Resources] CRITICAL for Java/Spring Boot apps to prevent OOM errors
      resources:
        requests: # The guaranteed minimum resources it needs to start
          cpu: "500m" # 0.5 CPU cores
          memory: "512Mi" # 512 Megabytes of RAM
        limits: # The absolute maximum it is allowed to consume
          cpu: "1000m" # 1 CPU core
          memory: "1Gi" # 1 Gigabyte of RAM. If it exceeds this, K8s kills it.

      # [5. Liveness and Readiness Probes] How K8s knows if your app is healthy
      livenessProbe: # If this fails, K8s restarts the container
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 15 # Wait 15s before checking (gives Spring Boot time to start)
        periodSeconds: 10 # Check every 10 seconds

      readinessProbe: # If this fails, K8s stops sending network traffic to it
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 15
```

**6. Essential `kubectl` Commands for Pods**

|              Command               |                                                                Explanation & Use Case                                                                 |                               Example                               |
| :--------------------------------: | :---------------------------------------------------------------------------------------------------------------------------------------------------: | :-----------------------------------------------------------------: |
|    `kubectl apply -f pod.yaml`     |                                                        Creates the Pod based on the YAML file.                                                        |                 `kubectl apply -f backend-pod.yaml`                 |
|         `kubectl get pods`         |                            Lists all Pods in the current namespace. Watch the `STATUS` column to see the lifecycle phases.                            | `kubectl get pods -w` (The `-w` flag watches for real-time changes) |
|   `kubectl describe pod <name>`    | Fetches the complete metadata, configuration, and **Events** log of the Pod. Use this immediately if your Pod is in a **Pending** or **Error** state. |               `kubectl describe pod library-api-pod`                |
|       `kubectl logs <name>`        |                           Prints the standard output/error (like Docker logs). Essential for debugging application crashes.                           |        `kubectl logs library-api-pod -f` (Streams the logs)         |
| `kubectl exec -it <name> -- <cmd>` |       Opens an interactive shell session inside the running Linux container. Useful for checking configurations or running local network tests.       |           `kubectl exec -it library-api-pod -- /bin/bash`           |
|    `kubectl delete pod <name>`     |                Destroys the Pod. (**Note:** If this Pod was created by a Deployment, a new one will instantly spin up to replace it).                 |                `kubectl delete pod library-api-pod`                 |

### ReplicaSet

**1. What is a ReplicaSet?**

As mentioned in the Pod section, Pods are ephemeral. If a worker node crashes, or if a Pod gets evicted due to lack of memory, that Pod is gone forever. Kubernetes does not automatically bring an unmanaged Pod back to life.

A **ReplicaSet (RS)** solves this problem. Its sole purpose is to maintain a stable set of replica Pods running at any given time. It acts as a "guardian" or a thermostat for your application. If you tell a ReplicaSet, "I always want 3 instances of my backend API running," it will constantly monitor the cluster to ensure exactly 3 instances exist.

**2. How Does It Work? (Labels and Selectors)**

A ReplicaSet does not strictly "own" Pods; it selects them based on **Labels**.

- The ReplicaSet uses a `selector` to count how many Pods currently exist with a specific label (e.g., `app: library-api`).
- If there are too few Pods (e.g., 2 instead of 3), the ReplicaSet uses its embedded Pod `template` to create a new one.
- If there are too many Pods (e.g., you manually created a Pod with the same label, making it 4), the ReplicaSet will ruthlessly terminate one of them to maintain the desired state of 3.

**3. Why We Rarely Use Them Directly**

**Crucial Note:** While understanding ReplicaSets is fundamental, **you will almost never create a ReplicaSet directly in a production environment**. Instead, you create a **Deployment**. A Deployment automatically creates and manages ReplicaSets for you, while adding the ability to do zero-downtime rolling updates (which ReplicaSets cannot do on their own).

### Deployment

**1. What is a Deployment?**

While a ReplicaSet ensures that a specific number of Pods are running, it has a major limitation: it does not handle application updates gracefully. If you change the Docker image version in a ReplicaSet YAML and apply it, the running Pods will not be updated. You would have to manually delete the old Pods so the ReplicaSet can recreate them with the new image.

A **Deployment** is a higher-level concept that manages ReplicaSets and provides declarative, zero-downtime updates to your application. It acts as the "manager" that dictates how and when your Pods should transition from one version to the next.

**2. Key Capabilities of a Deployment**

- **Rolling Updates (Zero Downtime):** When you release a new version of your application, the Deployment creates a new ReplicaSet. It then smoothly scales up the new ReplicaSet while simultaneously scaling down the old ReplicaSet. The application remains available to users throughout the entire process.
- **Rollbacks:** If a new deployment has a bug or crashes on startup, the Deployment keeps a history of your previous ReplicaSets. You can instantly rollback to a previous, stable version with a single command.
- **Pausing and Resuming:** You can pause a Deployment, make multiple configuration changes (like updating CPU limits, changing environment variables, and updating the image), and then resume it to apply all changes as a single rollout.
- **Scaling:** Like ReplicaSets, Deployments can be easily scaled up or down manually or via an autoscaler.

**3. The Rolling Update Strategy**

To achieve zero downtime, a Deployment uses two crucial mathematical parameters during an update:

- `maxSurge`: How many Pods can be created above the desired number of replicas during an update. (e.g., If `replicas=3` and `maxSurge=1`, the Deployment will temporarily run 4 Pods).
- `maxUnavailable`: How many Pods can be completely offline below the desired number of replicas during an update. (e.g., If `replicas=3` and `maxUnavailable=0`, there will always be at least 3 Pods running to serve traffic).

**4. YAML Specification (`deployment.yaml`)**

Continuing with the backend example, here is how you define a complete, production-ready Deployment for a Java Spring Boot application.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: library-api-deployment # The name of the Deployment
  labels:
    app: library-api
    env: production
spec:
  # [1. Replicas] The total number of Pods you want running
  replicas: 3

  # [2. Strategy] How the Deployment handles updates
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # Can spin up 1 extra Pod during rollout (Total: 4)
      maxUnavailable: 0 # Never drop below the 3 requested Pods

  # [3. Selector] How the Deployment finds the Pods it owns
  selector:
    matchLabels:
      app: library-api

  # [4. Pod Template] The exact blueprint for the Pods.
  # Note: The Deployment creates a ReplicaSet using this template.
  template:
    metadata:
      labels:
        app: library-api # Must match the selector above!
    spec:
      containers:
        - name: library-spring-boot
          image: myrepo/library-api:v2.0.0 # Updating this triggers a Rolling Update
          imagePullPolicy: Always

          ports:
            - containerPort: 8080
              protocol: TCP

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"

          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"

          # Probes are heavily relied upon by Deployments.
          # The Deployment will NOT route traffic to a new Pod or kill an old Pod
          # until the new Pod's readinessProbe passes!
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
```

**5. Essential `kubectl` Commands for Deployments**

Deployments introduce the `rollout` command suite, which is critical for managing the lifecycle of your application.

|                            Command                            |                                                             Explanation & Use Case                                                             |                                               Example                                               |
| :-----------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------: |
|              `kubectl apply -f deployment.yaml`               |                                                       Creates or updates the Deployment.                                                       |                               `kubectl apply -f library-deploy.yaml`                                |
|                   `kubectl get deployments`                   | Lists Deployments. Shows `READY` (current/desired), `UP-TO-DATE` (pods matching the current template), and `AVAILABLE` (pods serving traffic). |                                        `kubectl get deploy`                                         |
| `kubectl set image deployment <name> <container>=<new-image>` |                                Imperatively updates the Docker image, immediately triggering a rolling update.                                 | `kubectl set image deployment library-api-deployment library-spring-boot=myrepo/library-api:v2.1.0` |
|          `kubectl rollout status deployment <name>`           |            **Crucial:** Watches the real-time progress of a rolling update (e.g., “Waiting for replica set to finish scaling...”).             |                       `kubectl rollout status deploy library-api-deployment`                        |
|          `kubectl rollout history deployment <name>`          |                                       Shows a list of previous Revisions (versions) of this Deployment.                                        |                       `kubectl rollout history deploy library-api-deployment`                       |
|           `kubectl rollout undo deployment <name>`            |                          Instantly rolls back the Deployment to the previous working version if the new code crashes.                          |                        `kubectl rollout undo deploy library-api-deployment`                         |
| `kubectl rollout undo deployment <name> --to-revision=<num>`  |                                           Rolls back to a specific, older revision from the history.                                           |                `kubectl rollout undo deploy library-api-deployment --to-revision=2`                 |
|      `kubectl scale deployment <name> --replicas=<num>`       |                                                     Scales the number of Pods up or down.                                                      |                     `kubectl scale deploy library-api-deployment --replicas=5`                      |

### Service

**1. What is a Service?**

In Kubernetes, a Service is an abstract way to expose an application running on a set of Pods as a network service. It provides a stable, static IP address and a persistent DNS name for those Pods, acting as an internal load balancer.

**2. Why Do We Need Services? (The Problem it Solves)**

Pods are highly ephemeral (temporary). When a Deployment scales up or down, or when a Worker Node crashes, the old Pods are destroyed and new ones are created to replace them. Every time a new Pod is spun up, it is assigned a **brand new internal IP address**.

- **The Problem:** If your Frontend application is hardcoded to communicate with a specific Backend Pod's IP address, that connection will completely break as soon as the Backend Pod restarts or is replaced.
- **The Solution:** A Service provides a single, unchanging IP address that sits in front of your Backend Pods. The Frontend only needs to call the Service's IP or DNS name, and the Service will automatically route the traffic to the healthy, currently running Pods behind it.

**3. How Services Connect to Pods (Labels & Selectors)**

A Service does not directly link to specific Pods by their IDs or names. Instead, it uses **Label Selectors**.

The Service continuously scans the cluster for any Pods that have labels matching its `selector` field. The actual IP addresses of all matching, healthy Pods are automatically grouped and continuously updated in a list called **Endpoints**.

**4. Service Types**

Kubernetes provides four primary types of Services to handle different networking scenarios:

- **ClusterIP (Default):** Exposes the Service on an internal IP within the cluster. This makes the Service only reachable from inside the cluster. This is standard for Backend APIs or Databases that should not be exposed to the public internet.
- **NodePort:** Exposes the Service on each Worker Node's IP at a specific static port (ranging from 30000 to 32767). You can access the Service from outside the cluster by requesting `<NodeIP>:<NodePort>`.
- **LoadBalancer:** Integrates directly with a cloud provider's load balancer (e.g., AWS ALB/ELB, Google Cloud Load Balancing, Azure LB). K8s automatically provisions the underlying NodePort and ClusterIP, and the cloud provider assigns a public IP address to access the application from the internet.
- **ExternalName:** Maps the Service to a completely external DNS name (e.g., `api.external-service.com`) instead of using a typical selector. This is useful when your K8s application needs to talk to a legacy database hosted outside the cluster, but you want to use native K8s DNS resolution.

**5. YAML Specifications (`service.yaml`)**

```yaml
# ClusterIP Service for a Backend Application
apiVersion: v1
kind: Service
metadata:
  name: my-backend-service # The internal DNS name (e.g., http://my-backend-service)
  labels:
    tier: backend
spec:
  # [1. Type] Defaults to ClusterIP if omitted
  type: ClusterIP

  # [2. Selector] Finds and connects to Pods with these matching labels
  selector:
    app: backend-api

  # [3. Ports] Configures the network ports
  ports:
    - protocol: TCP
      port: 80 # The port the Service itself listens on
      targetPort: 8080 # The actual port the container is running on (e.g., Spring Boot on 8080)
```

```yaml
# NodePort Service for a Frontend Application
apiVersion: v1
kind: Service
metadata:
  name: my-frontend-service
spec:
  # Changes the type to NodePort to allow external access
  type: NodePort

  selector:
    app: frontend-web

  ports:
    - protocol: TCP
      port: 80 # The Service's internal port
      targetPort: 80 # The container's port (e.g., Nginx)
      nodePort: 30050 # (Optional) Forces K8s to open this specific port on the Nodes. If omitted, K8s picks a random one.
```

**6. Essential `kubectl` Commands for Services**

|                          Command                          |                                                                       Explanation & Use Case                                                                       |                        Example                        |
| :-------------------------------------------------------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------: | :---------------------------------------------------: |
|              `kubectl apply -f service.yaml`              |                                                       Creates or updates the Service from the YAML manifest.                                                       |         `kubectl apply -f frontend-svc.yaml`          |
|                     `kubectl get svc`                     |                        Lists the Services in the namespace. Shows crucial info like **CLUSTER-IP**, **EXTERNAL-IP**, and open **PORT(S)**.                         |                   `kubectl get svc`                   |
|               `kubectl describe svc <name>`               | Shows detailed configuration. **Crucial for debugging:** Check the **Endpoints** row. If it says `<none>`, your Service selector is not matching any running Pods. |       `kubectl describe svc my-backend-service`       |
|     `kubectl get endpoints <name>` (`kubectl get ep`)     |                                Directly lists the actual IP addresses of the Pods that the Service is currently routing traffic to.                                |          `kubectl get ep my-backend-service`          |
| `kubectl port-forward svc/<name> <local_port>:<svc_port>` |                Opens a secure network tunnel from your local machine to the Service. Excellent for quick local testing without Ingress or NodePort.                | `kubectl port-forward svc/my-backend-service 8080:80` |
|                `kubectl delete svc <name>`                |                       Deletes the Service. (**Note:** This only removes network routing; it does **not** stop or delete the Pods behind it).                       |        `kubectl delete svc my-backend-service`        |

### Namespace

**1. What is a Namespace?**

In Kubernetes, a **Namespace** provides a mechanism for isolating groups of resources within a single physical cluster. You can think of Namespaces as "virtual clusters" running on top of the same physical hardware.

- **Analogy:** Think of a Namespace like a folder on your computer's operating system. You cannot have two files named `app-config.txt` in the same folder, but you can easily have them in two different folders. Similarly, you cannot have two Pods named `backend-api` in the same Namespace, but you can have one in the `dev` Namespace and one in the `prod` Namespace.

**2. Why Use Namespaces? (Core Benefits)**

If you are running a small, personal project (like your local Minikube setup), the `default` namespace is usually enough. However, in enterprise environments, Namespaces are critical:

- **Environment Separation:** You can use the same cluster to host `development`, `staging`, and `production` environments by separating them into different Namespaces.
- **Multi-Tenancy (Team Isolation):** If multiple teams share a cluster, giving each team their own Namespace prevents Team A from accidentally deleting or modifying Team B's applications.
- **Resource Quotas:** You can apply memory and CPU limits to an entire Namespace. For example, you can ensure the `dev` Namespace cannot consume more than 20% of the cluster's total RAM, protecting the `prod` Namespace from resource starvation.
- **Access Control (RBAC):** You can grant a user "Admin" rights strictly within the `dev` Namespace, but only "View" rights in the `prod` Namespace.

**3. The Built-in Namespaces**

When you first start a Kubernetes cluster, it automatically creates several standard Namespaces. You will see these when you run `kubectl get ns`:

- `default`: The standard playground. If you deploy a Pod or Service without specifying a Namespace, it goes here.
- `kube-system`: The most critical Namespace. This is where Kubernetes runs its own internal components (like the API Server, `etcd`, `kube-dns`, and network plugins). **Rule of thumb:** Never manually create or delete resources in this Namespace.
- `kube-public`: A Namespace readable by all users (even unauthenticated ones), typically used for cluster bootstrap information.
- `kube-node-lease`: Used by Kubernetes for node heartbeat data to determine if a Worker Node has failed.

**4. Scoped vs. Cluster-Wide Resources**

It is important to understand that most resources are tied to a Namespace, but not all.

- **Namespaced Resources:** Pods, Deployments, ReplicaSets, Services, ConfigMaps, Secrets, PVCs (Persistent Volume Claims).
- **Cluster-Scoped Resources**: Nodes, Persistent Volumes (PVs), StorageClasses, and Namespaces themselves. (You cannot put a Node inside a Namespace).

**5. Cross-Namespace Communication (DNS Routing)**

Resources can communicate across Namespaces! If a Frontend Pod in the `default` namespace wants to talk to a Database Service in the `prod` namespace, it cannot simply call `http://database-service`.
It must use the **Fully Qualified Domain Name (FQDN)** provided by Kubernetes DNS:
`http://<service-name>.<namespace-name>.svc.cluster.local` (Example: `http://database-service.prod.svc.cluster.local`)

**6. YAML Specifications**

- **Creating the Namespace (`namespace.yaml`)**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging-env
  labels:
    team: backend-engineers
```

- **Deploying Resources INTO a Namespace:** To put a resource into a specific Namespace, you must explicitly declare it in the `metadata.namespace` field of that resource's YAML file. If you omit this, K8s puts it in `default`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: library-api
  namespace: staging-env # THIS puts the Deployment in the staging Namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: library
  template:
    metadata:
      labels:
        app: library
    spec:
      containers:
        - name: spring-boot
          image: myrepo/library-api:v1.0
```

**7. Essential `kubectl` Commands for Namespaces**

Working with Namespaces requires you to heavily use the `-n` (or `--namespace`) flag in `kubectl`. If you forget `-n`, `kubectl` will only look in the `default` namespace and might tell you "Resource not found".

|                          Command                          |                                              Explanation & Use Case                                              |                          Example                           |
| :-------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------: | :--------------------------------------------------------: |
|        `kubectl get namespaces` (`kubectl get ns`)        |               Lists all Namespaces in the cluster and their current status (Active / Terminating).               |                      `kubectl get ns`                      |
|             `kubectl create namespace <name>`             |                        Imperatively creates a new Namespace without needing a YAML file.                         |                `kubectl create ns dev-env`                 |
|       `kubectl apply -f <file.yaml> -n <namespace>`       | Deploys the resources in the YAML file directly into the specified Namespace (overriding the file if necessary). |           `kubectl apply -f app.yaml -n dev-env`           |
|             `kubectl get pods -n <namespace>`             |                               Lists Pods strictly within the specified Namespace.                                |             `kubectl get pods -n kube-system`              |
|         `kubectl get all -A` (`--all-namespaces`)         |   Lists **every resource across ALL Namespaces**. Extremely useful when debugging or locating lost resources.    |                    `kubectl get all -A`                    |
|                `kubectl delete ns <name>`                 |    **DANGER:** Deletes the Namespace and **permanently destroys** every Pod, Service, and resource inside it.    |                `kubectl delete ns dev-env`                 |
| `kubectl config set-context --current --namespace=<name>` |      **Pro Tip:** Sets a default Namespace for your current context so you don’t need to keep typing `-n`.       | `kubectl config set-context --current --namespace=dev-env` |

## 7. Configuration Management: ConfigMaps and Secrets

### ConfigMaps (Non-Sensitive Data)

**1. What is a ConfigMap?**

A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

- **Use Cases:** Database URLs, feature flags, application port numbers, plain-text configuration files (like `application.properties` or `nginx.conf`).

**2. YAML Specification (`configmap.yaml`)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: library-api-config # The name referenced by the Pod
  namespace: default
data:
  # Key-Value pairs
  DB_HOST: "postgres-svc.database.svc.cluster.local"
  DB_PORT: "5432"
  SPRING_PROFILES_ACTIVE: "prod"

  # You can also store entire multi-line files as a value!
  application.properties: |
    server.port=8080
    logging.level.root=INFO
    library.feature-flag.new-ui=true
```

### Secrets (Sensitive Data)

**1. What is a Secret?**

A Secret is an object that contains a small amount of sensitive data such as passwords, OAuth tokens, or SSH keys. Putting this information in a Secret is safer and more flexible than putting it verbatim in a Pod definition or a container image.

**Crucial Security Note:** By default, Kubernetes Secrets are **NOT encrypted** in the YAML file or in the `etcd` database; they are merely **Base64 encoded**. Base64 is an encoding mechanism, not encryption. Anyone who can read the Secret object can decode it. (In production, you must enable `etcd` encryption at rest or use external tools like HashiCorp Vault or AWS Secrets Manager).

**2. YAML Specification (`secret.yaml`)**

To write a Secret YAML, you must first Base64 encode your values. (In Linux/macOS: `echo -n 'my-super-secret-password' | base64` -> `bXktc3VwZXItc2VjcmV0LXBhc3N3b3Jk`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: library-db-credentials
type: Opaque # Opaque means arbitrary user-defined key/value pairs
data: # ALL values here MUST be Base64 encoded
  username: YWRtaW4= # Decodes to: admin
  password: bXktc3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

### Injecting Configs & Secrets into a Pod/Deployment

This is where the magic happens. Here is how you modify your Deployment's `spec.template.spec` to use the ConfigMap and Secret we just created.

**Injection Methods**

- **As Environment Variables:** Best for simple key-value pairs.
- **As Mounted Volumes:** Best for injecting entire files (like `.properties` or `.conf` files).

```yaml
# Inside your Deployment's Pod template...
spec:
  containers:
    - name: library-spring-boot
      image: myrepo/library-api:v2.0.0

      # --- METHOD 1: Injecting as Environment Variables ---
      env:
        # A. Injecting a specific key from a ConfigMap
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: library-api-config
              key: DB_HOST

        # B. Injecting a specific key from a Secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: library-db-credentials
              key: password

      # C. Injecting ALL keys from a ConfigMap as env vars at once
      envFrom:
        - configMapRef:
            name: library-api-config

      # --- METHOD 2: Injecting as a Mounted Volume ---
      volumeMounts:
        - name: app-config-volume # Must match the volume name below
          mountPath: /app/config # Where the file will appear inside the Linux container
          readOnly: true

  # Defining the Volumes that the container can mount
  volumes:
    - name: app-config-volume
      configMap:
        name: library-api-config # Mounts the 'application.properties' file here
```

### Essential `kubectl` Commands for ConfigMaps & Secrets

|                               Command                               |                                                  Explanation & Use Case                                                  |                                    Example                                    |
| :-----------------------------------------------------------------: | :----------------------------------------------------------------------------------------------------------------------: | :---------------------------------------------------------------------------: |
|   `kubectl create configmap <name> --from-literal=<key>=<value>`    |                              Imperatively creates a ConfigMap quickly without a YAML file.                               |             `kubectl create cm app-config --from-literal=ENV=dev`             |
|        `kubectl create configmap <name> --from-file=<path>`         |                      Creates a ConfigMap from an existing local file (e.g., an Nginx config file).                       |            `kubectl create cm nginx-conf --from-file=./nginx.conf`            |
| `kubectl create secret generic <name> --from-literal=<key>=<value>` | Imperatively creates a Secret. **Tip:** Often preferred over YAML because kubectl automatically handles Base64 encoding. | `kubectl create secret generic db-pass --from-literal=password='P@ssw0rd123'` |
|              `kubectl get cm` / `kubectl get secrets`               |                                Lists the existing ConfigMaps or Secrets in the namespace.                                |                             `kubectl get secrets`                             |
|                    `kubectl describe cm <name>`                     |                                  Shows the contents of a ConfigMap (plain-text values).                                  |                   `kubectl describe cm library-api-config`                    |
|                  `kubectl describe secret <name>`                   |                        Shows the keys inside a Secret, but hides the actual values for security.                         |               `kubectl describe secret library-db-credentials`                |
|                 `kubectl get secret <name> -o yaml`                 |                         Outputs the Secret in YAML format, revealing the Base64-encoded strings.                         |              `kubectl get secret library-db-credentials -o yaml`              |

## 8. Storage: Persistent Volumes (PV) & Persistent Volume Claims (PVC)

### The Ephemeral Storage Problem

By design, containers in Kubernetes are ephemeral (temporary). If a container crashes or a Pod is deleted during an update, Kubernetes will spin up a new one, but **any data generated inside that container is completely wiped out.**

While this is perfectly fine for stateless applications (like a React frontend or a Spring Boot API), it is disastrous for stateful applications (like MySQL or PostgreSQL databases) where data loss is unacceptable. Kubernetes solves this by decoupling the lifecycle of storage from the lifecycle of the Pod using the PV and PVC architecture.

### What is a Persistent Volume (PV)?

A Persistent Volume (PV) is an actual piece of physical or virtual storage in the cluster. It is either provisioned manually by a cluster administrator or provisioned dynamically by the system using a `StorageClass`.

- **Examples:** An NFS (Network File System) share, a cloud provider's disk (like AWS EBS, GCP Persistent Disk, or Azure Disk), or even a local directory on a Worker Node.
- **Core Concept:** A PV is a cluster-level resource (it does not belong to any specific Namespace). Its lifecycle is entirely independent of any individual Pod. If a Pod dies, the PV and its data remain intact.

### What is a Persistent Volume Claim (PVC)?

A Persistent Volume Claim (PVC) is a "request for storage" made by a developer or an application.

Instead of a Pod connecting directly to a physical hard drive, the Pod submits a request via a PVC: "I need a 5GB volume with read/write access." When you create a PVC, Kubernetes automatically looks for an available PV that satisfies those requirements (size and access mode) and **binds** them together in a 1-to-1 relationship. The Pod then simply mounts the PVC as a standard volume.

### Access Modes

When defining PVs and PVCs, you must specify how the storage volume can be mounted to the Worker Nodes. The three primary modes are:

- **`ReadWriteOnce` (RWO):** The volume can be mounted as read-write by a single Node. (Standard for most databases).
- **`ReadOnlyMany` (ROX):** The volume can be mounted as read-only by many Nodes simultaneously.
- **`ReadWriteMany` (RWX):** The volume can be mounted as read-write by many Nodes simultaneously. (Requires specific network storage solutions like NFS or AWS EFS).

### YAML Specifications

**Example 1: Creating a Persistent Volume (PV)**

(**Note:** In modern cloud environments, PVs are usually created automatically via Dynamic Provisioning, but creating one manually is essential for local testing like Minikube).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manual-pv
spec:
  capacity:
    storage: 10Gi # Allocates 10 Gigabytes of storage
  accessModes:
    - ReadWriteOnce # Only one Node can mount this for reading/writing
  persistentVolumeReclaimPolicy: Retain # When the PVC is deleted, Retain the data on the PV
  hostPath: # Uses a directory on the Worker Node (Great for Minikube testing)
    path: "/mnt/data/mysql"
```

**Example 2: Creating a Persistent Volume Claim (PVC)**

(This is the file the developer writes to request the storage).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-database-pvc # The name of the storage claim
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce # Must match the Access Mode of an available PV
  resources:
    requests:
      storage: 5Gi # Requesting 5GB of storage
```

**Example 3: Mounting the PVC into a Pod**

To make a Pod store data permanently, you must modify the Pod's `spec` to use the PVC.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-database-pod
spec:
  containers:
    - name: mysql-container
      image: mysql:8.0
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: "mypassword"

      # [1. Mount the volume inside the Container]
      volumeMounts:
        - name: database-storage # This name must perfectly match the volume name below
          mountPath: /var/lib/mysql # The internal path where MySQL expects to save its data

  # [2. Define the Volume and link it to the PVC]
  volumes:
    - name: database-storage
      persistentVolumeClaim:
        claimName: mysql-database-pvc # Points exactly to the PVC we created in Example 2
```

### Essential `kubectl` Commands for Storage

|            Command             |                                                                  Explanation & Use Case                                                                   |                  Example                  |
| :----------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------: | :---------------------------------------: |
| `kubectl apply -f <file.yaml>` |                                                       Creates the PV or PVC from the YAML manifest.                                                       |     `kubectl apply -f mysql-pvc.yaml`     |
|        `kubectl get pv`        |                          Lists all **PersistentVolumes** in the cluster. Shows total capacity, access modes, and the bound PVC.                           |             `kubectl get pv`              |
|       `kubectl get pvc`        |       Lists all **PersistentVolumeClaims** in the current Namespace. **Crucial:** If `STATUS` is `Bound`, the PVC is successfully attached to a PV.       |             `kubectl get pvc`             |
| `kubectl describe pvc <name>`  | Shows detailed information about the PVC. Extremely useful when a PVC is stuck in `Pending` (often due to insufficient space or mismatched access modes). | `kubectl describe pvc mysql-database-pvc` |
|  `kubectl delete pvc <name>`   |            Deletes the storage claim. Whether the underlying data is deleted or retained depends on the PV’s `persistentVolumeReclaimPolicy`.             |  `kubectl delete pvc mysql-database-pvc`  |

## 9. Ingress: The Smart Router (Layer 7 Load Balancing)

### What is an Ingress?

Up to this point, we used a `Service` (like `NodePort` or `LoadBalancer`) to expose applications to the outside world. However, Services operate at Layer 4 (Transport Layer - TCP/UDP). They only understand IP addresses and Ports.

An **Ingress** operates at Layer 7 (Application Layer - HTTP/HTTPS). It is an API object that acts as a smart router or reverse proxy for your cluster. Instead of creating a separate, expensive Cloud Load Balancer for every single Service you deploy, you can use one Ingress to route traffic to dozens of different backend Services based on the URL path or the domain name requested by the user.

**Why use it?**

Instead of using 10 LoadBalancers for 10 Services (expensive), you use 1 Ingress Controller to route traffic based on domains (e.g., `api.example.com` -> Service A, `web.example.com` -> Service B).

### The Ingress Controller

**Crucial Note:** Creating an Ingress YAML file only defines the routing rules. It does **absolutely nothing** on its own.

To make the rules work, you must have an **Ingress Controller** running in your cluster. The controller is an application (usually a Pod itself) that reads the Ingress rules and implements them.

- **Popular Controllers:** NGINX Ingress Controller (the most common), Traefik, HAProxy, or cloud-native ones like AWS ALB Ingress Controller.
- **In Minikube:** You enable it simply by running: `minikube addons enable ingress`.

### Key Capabilities of Ingress

**1. Path-Based Routing (Simple Fanout)**

**Concept:** Path-based routing (also known as a simple fanout) allows you to route traffic from a single IP address and a single domain name to multiple backend Services based on the HTTP URI (the path after the domain).

**Use case:** You have one domain `example.com`. You want `example.com/api` to hit your backend Java Spring Boot service, and `example.com/web` to hit your frontend React service.

**YAML Specification (`path-based-ingress.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    # Often needed if your backend doesn't expect the /api prefix
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com # The single domain name
      http:
        paths:
          # Rule 1: Traffic to /api goes to the backend service
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-api-service
                port:
                  number: 8080

          # Rule 2: Traffic to /web goes to the frontend service
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: frontend-web-service
                port:
                  number: 80
```

- **Explanation:** The Ingress controller evaluates the paths in order. Because `pathType` is set to `Prefix`, a request to `myapp.example.com/api/v1/users` will match the `/api` rule and be forwarded to the `backend-api-service` on port `8080`.

**2. Host-Based Routing (Name-Based Virtual Hosting)**

**Concept:** Host-based routing supports routing HTTP traffic to multiple hostnames (domains or subdomains) using the same Ingress IP address. The Ingress controller inspects the HTTP `Host` header of the incoming request to determine which backend Service should receive it.

**Use case:** You have two completely separate applications sharing the same cluster. `api.example.com` needs to go to Service A, and `admin.example.com` needs to go to Service B.

**YAML Specification (`host-based-ingress.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    # Host 1: API Subdomain
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080

    # Host 2: Admin Subdomain
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

- **Explanation:** In this `spec`, there are two separate items in the `rules` array, each defining a different `host`. If a user types `api.example.com` into their browser, the DNS resolves to the Ingress controller's IP. The controller sees the `Host: api.example.com` header and routes the traffic exclusively to the `api-service`.

**3. TLS (HTTPS Setup)**

**Concept:** By default, Ingress traffic is unencrypted (HTTP). You can secure an Ingress by specifying a `tls` section containing a Secret that holds a TLS private key and certificate. This enables "TLS Termination" at the Ingress controller - meaning the external connection is encrypted (HTTPS), but the internal traffic between the controller and your Pods is usually plain HTTP, saving processing power on your application containers.

**Prerequisite:** Before applying the Ingress, you must create a Kubernetes Secret of type `tls`.

```bash
# Command to create a TLS secret from your certificate files
kubectl create secret tls myapp-tls-secret --cert=path/to/tls.crt --key=path/to/tls.key
```

**YAML Specification (`tls-ingress.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-secured-ingress
spec:
  ingressClassName: nginx

  # [NEW] TLS Configuration Block
  tls:
    - hosts:
        - secure.example.com # The domain this certificate applies to
      secretName: myapp-tls-secret # The K8s Secret containing the .crt and .key

  # Standard Routing Rules
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-app-service
                port:
                  number: 80
```

- **Explanation:** The `tls` array specifies which `hosts` should be secured using which `secretName`.
  - When the Ingress controller reads this, it automatically configures itself to listen on port 443 (HTTPS) for `secure.example.com` and uses the provided certificate to encrypt the connection.
  - **Note:** You can combine all three of these! You can have an Ingress that uses TLS, has multiple Hosts, and uses Path-based routing under each Host.

### Essential `kubectl` Commands for Ingress

| Command                                          | Explanation & Use Case                                                                                                                     | Example                                                      |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| `kubectl apply -f ingress.yaml`                  | Applies the routing rules to the cluster.                                                                                                  | `kubectl apply -f library-ingress.yaml`                      |
| `kubectl get ingress` (`kubectl get ing`)        | Lists all Ingress resources. **Important:** Wait for the `ADDRESS` column to show an IP - this is the IP you map your DNS (`A Record`) to. | `kubectl get ing`                                            |
| `kubectl describe ingress <name>`                | Shows the detailed routing table. Excellent for debugging to verify paths are mapped to the correct backend Services.                      | `kubectl describe ing library-system-ingress`                |
| `kubectl delete ingress <name>`                  | Deletes the routing rules. Services and Pods continue running, but external access via these rules stops.                                  | `kubectl delete ing library-system-ingress`                  |
| `kubectl logs -n ingress-nginx <nginx-pod-name>` | **Advanced debugging:** Inspect logs of the NGINX Ingress Controller Pod if routing is not working or configs are rejected.                | `kubectl logs -n ingress-nginx ingress-nginx-controller-xyz` |

## 10. Helm (The Package Manager)

### What is Helm?

Just as Ubuntu has `apt`, CentOS has `yum`, and Java has `Maven`, Kubernetes has `Helm`. Helm is the official package manager for Kubernetes. It simplifies the process of defining, installing, and upgrading even the most complex Kubernetes applications.

**Install Helm:**

```bash
# 1. Download the installation script
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

# 2. Assign execution permissions to the script
chmod 700 get_helm.sh

# 3. Run the script to install Helm
./get_helm.sh
```

### The Problem Helm Solves

Imagine you are deploying a complete application - for instance, a comprehensive Library Management System. To run this in production, you might need:

- A Deployment for the Java Spring Boot backend.
- A Service (ClusterIP) for the backend.
- A StatefulSet and PVCs for a PostgreSQL database.
- A Secret for the database passwords.
- A ConfigMap for application properties.
- An Ingress to route external traffic.

Without Helm, you have to manually manage, update, and run `kubectl apply -f` on 6 or more static YAML files. If you want to deploy a second instance of this system for a staging environment, you have to copy all those files and manually find-and-replace every name, label, and namespace.

Helm solves this by introducing **Templating**. Instead of hardcoding values into static YAML files, Helm allows you to create dynamic blueprints. You define variables in a single file, and Helm generates all the necessary Kubernetes YAML files and deploys them for you.

### The Core Concepts of Helm

Helm relies on three big concepts:

1. **Chart:** A Helm package. It is a bundle of information necessary to create an instance of a Kubernetes application. Think of it as a template or a blueprint.
2. **Repository (Repo):** A location where packaged charts can be stored and shared (similar to Docker Hub for images or Maven Central for Java dependencies). Artifact Hub is the most popular public repository.
3. **Release:** A running instance of a chart in a Kubernetes cluster. You can install the exact same chart multiple times in the same cluster, and each time it will create a new, uniquely named Release.

### The Anatomy of a Helm Chart

If you create a Helm chart (using `helm create my-app`), Helm generates a specific directory structure. Here is what you need to understand:

```text
my-app/
├── Chart.yaml          # Metadata about the chart (name, version, description)
├── values.yaml         # The DEFAULT configuration variables for this chart
├── charts/             # Dependencies (if this chart relies on other charts, like a DB)
└── templates/          # The actual Kubernetes YAML files, heavily using Go templates
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

**The Magic of `values.yaml`:** This is the most important file for a user. Instead of editing `deployment.yaml` directly, you just change a variable in `values.yaml`.

- Example in `values.yaml`: `replicaCount: 3`
- Example in `templates/deployment.yaml`: `replicas: {{ .Values.replicaCount }}`
- When Helm runs, it injects the `3` into the final YAML sent to Kubernetes.

### Essential Helm Commands

|                       Command                        |                                                 Explanation & Use Case                                                 |                              Example                              |
| :--------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------: | :---------------------------------------------------------------: |
|             `helm repo add <name> <url>`             |                 Adds a public repository to your local Helm client so you can download charts from it.                 |    `helm repo add bitnami https://charts.bitnami.com/bitnami`     |
|             `helm search repo <keyword>`             |                             Searches added repositories for a specific application chart.                              |                     `helm search repo mysql`                      |
|        `helm install <release-name> <chart>`         |                             Installs a chart into the cluster, creating a new **Release**.                             |             `helm install my-database bitnami/mysql`              |
| `helm install <release-name> <chart> -f values.yaml` | **Crucial:** Installs a chart using a custom `values.yaml` to override default settings (passwords, ports, resources). | `helm install my-database bitnami/mysql -f my-custom-values.yaml` |
|               `helm list` (`helm ls`)                |                               Lists all running Helm Releases in the current namespace.                                |                            `helm list`                            |
|        `helm upgrade <release-name> <chart>`         |                Upgrades an existing release to a new chart version or applies new configuration values.                |    `helm upgrade my-database bitnami/mysql -f new-values.yaml`    |
|            `helm history <release-name>`             |                                   Shows the revision history of a specific release.                                    |                    `helm history my-database`                     |
|         `helm rollback <release> <revision>`         |                 Instantly rolls back an application to a previous stable revision if an upgrade fails.                 |                   `helm rollback my-database 1`                   |
|           `helm uninstall <release-name>`            |                     Completely deletes the release and **all Kubernetes resources** created by it.                     |                   `helm uninstall my-database`                    |

### A Practical Example: Installing NGINX with Helm

**Step 1: Add a popular repository**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

**Step 2: Install NGINX with a custom variable**

Instead of writing a Deployment and a Service YAML, we just pass a variable (`--set`) to change the load balancer IP type or replica count.

```bash
# This single command downloads the chart, generates the YAML, and deploys it.
helm install my-web-server bitnami/nginx --set replicaCount=2
```

**Step 3: Verify the deployment**

```bash
# See the Helm release
helm list

# See the underlying K8s resources Helm automatically created for you!
kubectl get pods,svc
```

## 11. StatefulSets: Managing Stateful Applications

### What is a StatefulSet?

Throughout this tutorial, we have used **Deployments** to manage our Pods. Deployments are designed for **stateless** applications (like a Spring Boot web server or a React frontend). In a Deployment, every Pod is an exact, interchangeable clone. If `frontend-pod-a7x9` crashes, Kubernetes replaces it with `frontend-pod-b2y4`, and the application doesn't care.

However, **Stateful** applications (like databases: MySQL, MongoDB, Kafka, or Elasticsearch) care deeply about their identity and their data.

- If you run a primary-replica database cluster, the replica must know the exact network address of the primary node to sync data.
- If a database Pod restarts, it must reconnect to the exact same persistent storage volume it was using before it crashed.

A **StatefulSet** is the Kubernetes workload API object used to manage these stateful applications. It manages the deployment and scaling of a set of Pods, and provides guarantees about the **ordering and uniqueness** of these Pods.

### Key Features of a StatefulSet (Deployment vs. StatefulSet)

- **Predictable, Ordered Pod Names:**
  - `Deployment`: Pods get random hashes (`my-app-7b9c4d...`).
  - `StatefulSet`: Pods get sticky, sequential, numbered names (`my-db-0`, `my-db-1`, `my-db-2`).
- **Stable Network Identity:**
  - Every Pod in a StatefulSet gets its own persistent DNS hostname. Even if `my-db-0` crashes and is recreated on a different Worker Node, its DNS name remains exactly the same, allowing other Pods to reconnect to it reliably.
- **Stable, Dedicated Storage (`volumeClaimTemplates`):**
  - `Deployment`: All Pods usually share the same PVC (if configured), or lose their data.
  - `StatefulSet`: You do not create a single PVC. Instead, you provide a `volumeClaimTemplate`. When K8s spins up `my-db-0`, it dynamically creates a brand new PVC just for `my-db-0`. When it spins up `my-db-1`, it creates a separate PVC just for `my-db-1`. If `my-db-0` dies, K8s ensures the replacement Pod is reattached to that exact same PVC.
- **Ordered Deployment and Scaling:**
  - When you scale a StatefulSet to 3 replicas, it doesn't create them all at once. It starts `my-db-0`. It waits for `my-db-0` to be fully `Running` and `Ready`. Only then does it start `my-db-1`, and so on.
  - When scaling down, it deletes them in reverse order (e.g., `my-db-2` is terminated before `my-db-1`).

### The Prerequisite: The Headless Service

To give each Pod in a StatefulSet a stable network identity, you **must** create a "Headless Service."
A Headless Service is simply a standard Kubernetes Service, but you set `clusterIP: None`. This tells Kubernetes: "Do not load-balance traffic across these Pods. Instead, create a DNS record for every single Pod so they can be addressed individually."

### YAML Specification (`statefulset.yaml`)

A complete StatefulSet deployment requires two parts in the YAML file: The Headless Service and the StatefulSet itself.

```yaml
# --- Part 1: The Headless Service ---
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless-svc # The name used to generate the DNS records
  labels:
    app: mysql
spec:
  clusterIP: None # CRITICAL: This makes it "Headless"
  ports:
    - port: 3306
      name: mysql
  selector:
    app: mysql

# --- Part 2: The StatefulSet ---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-db # The base name for the Pods (mysql-db-0, mysql-db-1...)
spec:
  serviceName: "mysql-headless-svc" # CRITICAL: Links the StatefulSet to the Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql

  # [1. The Pod Template]
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-data # Must match the template name below
              mountPath: /var/lib/mysql

  # [2. Volume Claim Templates - The Magic of StatefulSets]
  # K8s will automatically create a PVC based on this template for EVERY Pod.
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

**How to connect to these Pods:**

Once deployed, other applications in the cluster can reach specific database nodes using their stable DNS names:

- `mysql-db-0.mysql-headless-svc.default.svc.cluster.local` (Usually configured as the Primary/Master node)
- `mysql-db-1.mysql-headless-svc.default.svc.cluster.local` (Usually a Read-Replica)

### Essential `kubectl` Commands for StatefulSets

|                    Command                     |                                                Explanation & Use Case                                                |                  Example                  |
| :--------------------------------------------: | :------------------------------------------------------------------------------------------------------------------: | :---------------------------------------: |
|      `kubectl apply -f statefulset.yaml`       |                                Creates the **Headless Service** and the StatefulSet.                                 |      `kubectl apply -f db-sts.yaml`       |
| `kubectl get statefulsets` (`kubectl get sts`) |                                        Lists the StatefulSets in the cluster.                                        |             `kubectl get sts`             |
|             `kubectl get pods -w`              | **Pro Tip:** Watch Pods start **one by one in order** (0 → 1 → 2). Extremely useful to verify StatefulSet behavior.  |    `kubectl get pods -l app=mysql -w`     |
|               `kubectl get pvc`                |          Views the automatically generated PVCs (e.g. `mysql-data-mysql-db-0`, `mysql-data-mysql-db-1`, …).          |             `kubectl get pvc`             |
|  `kubectl scale sts <name> --replicas=<num>`   |                   Scales the StatefulSet. **Scaling down deletes the highest-numbered Pod first.**                   | `kubectl scale sts mysql-db --replicas=5` |
|          `kubectl delete sts <name>`           | Deletes the StatefulSet. **Important:** PVCs are **NOT** deleted automatically — this prevents accidental data loss. |       `kubectl delete sts mysql-db`       |

## 12. Security: Role-Based Access Control (RBAC)

### What is RBAC in Kubernetes?

By default, when you interact with a Kubernetes cluster using `kubectl`, you are likely using the `admin` credentials provided by Minikube or your cloud provider. This gives you absolute power to create, read, modify, or delete anything in the cluster.

In a real-world scenario, handing out cluster-admin access to every developer, CI/CD pipeline, and external tool is a massive security risk. **Role-Based Access Control (RBAC)** is the Kubernetes system that allows you to regulate exactly who can do what within the cluster, following the **Principle of Least Privilege**.

### The Three Pillars of RBAC

To understand RBAC, you only need to answer three questions:

1. **Who** is making the request? (_Subject_)
2. **What** permissions do they have? (_Role_)
3. **How** are they granted those permissions? (_RoleBinding_)

**Pillar 1: The Subjects (The "Who")**

A Subject is the entity trying to interact with the Kubernetes API. There are three types:

- **User Accounts:** Actual humans (e.g., `alice@company.com`). K8s does not manage users internally; it relies on external Identity Providers (like AWS IAM, Google OIDC, or Active Directory).
- **Groups:** A collection of users (e.g., `backend-developers`).
- **ServiceAccounts:** This is the most important concept for developers! A ServiceAccount is an identity created specifically for **applications and Pods** running inside the cluster. If your CI/CD pipeline or an ArgoCD agent needs to deploy an application, it uses a ServiceAccount.

**Pillar 2: Roles and ClusterRoles (The "What" / The Rules)**

A Role defines a set of rules representing a set of permissions. It dictates what Actions (Verbs) can be performed on what Objects (Resources).

- **Role (Namespace-Scoped):** Grants permissions only within a specific Namespace. (e.g., "Can read Pods in the `dev` namespace").
- **ClusterRole (Cluster-Scoped):** Grants permissions across the entire cluster, or to cluster-level resources like Nodes and PersistentVolumes.

**The Rules Mapping:**

- **Verbs (Actions):** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`.
- **Resources (Objects):** `pods`, `deployments`, `services`, `secrets`, etc.

**Pillar 3: RoleBindings and ClusterRoleBindings (The "How" / The Link)**

A Role is just a list of rules; it does nothing on its own. A **Binding** is what attaches a Role (the rules) to a Subject (the user or ServiceAccount).

- **RoleBinding:** Binds a Role (or ClusterRole) to a Subject within a specific Namespace.
- **ClusterRoleBinding:** Binds a ClusterRole to a Subject across the entire cluster.

### YAML Specifications (A Practical Example)

Let's say you are building a Library Management System. You want to create a dedicated CI/CD pipeline that has permission to deploy and update the Spring Boot backend, but only within the `library-dev` namespace. It should not be able to delete databases or touch the `production` namespace.

**Step 1: Create the ServiceAccount (`service-account.yaml`)**

This is the identity your CI/CD pipeline will use.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: library-ci-pipeline
  namespace: library-dev
```

**Step 2: Create the Role (`role.yaml`)**

Here we define the exact permissions. Notice how we restrict the verbs and resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: library-developer-role
  namespace: library-dev
rules:
  # Rule 1: Can view, create, and update Deployments and StatefulSets
  - apiGroups: ["apps"] # The API group the resources belong to
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  # Rule 2: Can view and create Pods and Services, but CANNOT delete them
  - apiGroups: [""] # The core API group is represented by an empty string
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create"]

  # Rule 3: STRICT RESTRICTION - Can only view Secrets, cannot create or modify them
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
```

**Step 3: Create the RoleBinding (`role-binding.yaml`)**

This links the `library-ci-pipeline` ServiceAccount to the `library-developer-role`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: library-ci-binding
  namespace: library-dev
subjects:
  # The "Who"
  - kind: ServiceAccount
    name: library-ci-pipeline
    namespace: library-dev
roleRef:
  # The "What"
  kind: Role
  name: library-developer-role
  apiGroup: rbac.authorization.k8s.io
```

### Assigning a ServiceAccount to a Pod

If you have a Pod that needs to talk to the Kubernetes API (for example, if you wrote a custom application that needs to list other Pods), you must inject the ServiceAccount into the Pod's `spec`. If you don't, K8s assigns the highly restricted `default` ServiceAccount.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: internal-monitor-app
  namespace: library-dev
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: library-ci-pipeline # Injects the RBAC permissions into this Pod
      containers:
        - name: monitor
          image: myrepo/monitor:v1
```

### Essential `kubectl` Commands for RBAC (The Auth Tool)

Kubernetes provides an incredibly useful built-in tool called `auth can-i` to test permissions without having to actually run the commands and risk breaking things.

|                                       Command                                        |                                 Explanation & Use Case                                  |                                                        Example                                                        |
| :----------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------------------------------: |
|                        `kubectl auth can-i <verb> <resource>`                        |        Tests whether your **current user** has permission to perform an action.         |                   `kubectl auth can-i create deployments -n library-dev` _(returns `yes` or `no`)_                    |
| `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<namespace>:<name>` | **Crucial for testing:** Impersonates a ServiceAccount to verify its exact permissions. | `kubectl auth can-i delete secrets --as=system:serviceaccount:library-dev:library-ci-pipeline` _(should return `no`)_ |
|                   `kubectl get roles,rolebindings -n <namespace>`                    |            Lists all **Roles** and **RoleBindings** in a specific namespace.            |                                    `kubectl get roles,rolebindings -n library-dev`                                    |
|                    `kubectl describe role <name> -n <namespace>`                     |      Displays the actual RBAC rules (**verbs & resources**) defined inside a Role.      |                             `kubectl describe role library-developer-role -n library-dev`                             |

## 13. ArgoCD & GitOps: The Modern Deployment Standard

### The Concept of GitOps

**GitOps** is a set of practices that uses Git as the single source of truth for declarative infrastructure and applications.

- **The Old Way (CIOps / Push Model):** You write code, push it to GitHub. A Continuous Integration (CI) tool runs tests, builds a Docker image, and then runs `kubectl apply` to push the changes directly into the Kubernetes cluster. **The Problem:** Your CI tool needs admin access to your K8s cluster (a huge security risk), and if someone manually changes a Pod in K8s, Git has no idea.
- **The GitOps Way (Pull Model):** You store your Kubernetes YAML files (or Helm charts) in a Git repository. An agent (like ArgoCD) sits inside your Kubernetes cluster. It constantly watches that Git repository. If it notices a change in Git, it pulls the new YAML files and applies them to the cluster automatically.

Key Benefits of GitOps:

- **Single Source of Truth:** If it is not in Git, it does not exist in the cluster.
- **Version Control & Rollbacks:** If a deployment breaks the cluster, you don't run complicated K8s commands; you just click "Revert" on your Git commit, and the cluster instantly rolls back.
- **Enhanced Security:** Your CI pipeline only builds images and updates the Git repository. It never talks directly to Kubernetes.

### What is ArgoCD?

ArgoCD is a declarative, GitOps continuous delivery tool specifically built for Kubernetes. It is the agent that sits inside your cluster, constantly comparing the **Desired State** (what your YAML files in Git say) with the **Live State** (what is actually running in Kubernetes).

- **Drift Detection:** If a developer manually runs `kubectl scale deployment my-app --replicas=10` (Live State), but the Git repository says `replicas: 3` (Desired State), ArgoCD detects this as **"OutOfSync"** (Configuration Drift).
- **Self-Healing:** You can configure ArgoCD to automatically overwrite manual changes, instantly scaling it back down to 3 to match Git.

### Architecture & Core Concepts

When working with ArgoCD, you will encounter these specific terms:

- **Application:** A Custom Resource Definition (CRD) introduced by ArgoCD. It represents a deployed application and connects a specific Git repository folder to a specific Kubernetes Namespace.
- **Project:** A logical grouping of Applications. Used for multi-tenancy (e.g., restricting the "Backend Team" project so it can only deploy to the `backend-namespace`).
- **Sync:** The process of making the Live State match the Desired State. You can set this to `Manual` (you click a button to deploy) or `Automatic` (it deploys the second you merge a Git pull request).
- **Refresh:** ArgoCD polling the Git repository to check for the latest commits (happens automatically every 3 minutes by default).

### Installation Guide (Setting up ArgoCD in K8s)

Installing ArgoCD is surprisingly simple because it deploys itself as a set of standard Kubernetes resources.

**Step 1: Create the Namespace and Install ArgoCD**

```bash
# 1. Create a dedicated namespace for ArgoCD components
kubectl create namespace argocd

# 2. Apply the official ArgoCD installation manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Step 2: Verify the Installation**

Wait a few moments and verify that the ArgoCD pods are running.

```bash
kubectl get pods -n argocd
```

(You should see pods for the `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, etc.)

**Step 3: Access the ArgoCD Web UI**

By default, the ArgoCD API server is not exposed externally. For local development, use `port-forwarding`:

```bash
# Forward your local port 8080 to the ArgoCD server's HTTPS port 443
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open your browser and navigate to: [https://localhost:8080](https://localhost:8080) (Accept the self-signed certificate warning).

**Step 4: Get the Initial Admin Password**

To log in, the username is `admin`. ArgoCD automatically generates a secure password and stores it in a K8s Secret. Run this command to extract and decode it:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Using ArgoCD: Deploying an Application

There are two ways to deploy an application in ArgoCD: via the beautiful Web UI, or declaratively using a YAML file (The GitOps way). For your tutorial, the declarative way is the most important.

You define an ArgoCD `Application` resource. This tells ArgoCD exactly where to look for your code and where to deploy it.

**YAML Specification (`argo-application.yaml`):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-library-app
  namespace: argocd # The Application CRD must live in the argocd namespace
spec:
  # [1. Source] Where are the Kubernetes YAML files or Helm charts?
  source:
    repoURL: "https://github.com/your-username/my-k8s-manifests.git"
    targetRevision: HEAD # The Git branch (e.g., HEAD, main, or a specific tag)
    path: "apps/library-system" # The specific folder in the repo containing the YAMLs

  # [2. Destination] Which cluster and namespace should it be deployed to?
  destination:
    server: "https://kubernetes.default.svc" # The internal URL of the cluster ArgoCD is running in
    namespace: library-prod # The target namespace for your application pods

  # [3. Sync Policy] How should ArgoCD apply changes?
  syncPolicy:
    automated: # Enable automatic syncing when Git changes
      prune: true # Automatically delete resources in K8s if they are deleted in Git
      selfHeal: true # Automatically fix manual changes made directly via kubectl
    syncOptions:
      - CreateNamespace=true # Create the 'library-prod' namespace if it doesn't exist
```

**How to deploy this:**

- Push your actual application YAMLs (Deployment, Service, Ingress) to your Git repository.
- Run `kubectl apply -f argo-application.yaml`.
- ArgoCD immediately takes over. It connects to Git, reads your manifests, and deploys everything into the `library-prod` namespace.

### Essential ArgoCD CLI Commands

While the Web UI is fantastic for visualizing your deployments, ArgoCD also has a powerful CLI.

|             Command              |                                             Explanation & Use Case                                              |
| :------------------------------: | :-------------------------------------------------------------------------------------------------------------: |
|  `argocd login localhost:8080`   |                                 Authenticates your CLI with the ArgoCD server.                                  |
|        `argocd app list`         |            Lists all applications managed by ArgoCD along with their **Sync** and **Health** status.            |
| `argocd app sync my-library-app` |       Manually triggers a synchronization, forcing ArgoCD to pull the latest state from Git immediately.        |
| `argocd app diff my-library-app` | Shows the exact differences between the Git repository and the live Kubernetes cluster (similar to `git diff`). |

## References

- [Kubernetes in a Nutshell](https://medium.com/swlh/kubernetes-in-a-nutshell-tutorial-for-beginners-caa442dfd6c0)
- [Kubernetes Course](https://www.youtube.com/watch?v=X48VuDVv0do)
- [ArgoCD Course](https://www.youtube.com/watch?v=MeU5_k9ssrs)
