# Kubernetes Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Kubernetes](#introducing-kubernetes)
2. [Architecture & Components](#architecture--components)
3. [Installation & Setup](#installation--setup)
4. [kubectl Basics](#kubectl-basics)
5. [Pods](#pods)
6. [ReplicaSets](#replicasets)
7. [Deployments](#deployments)
8. [Services](#services)
9. [Namespaces](#namespaces)
10. [ConfigMaps & Secrets](#configmaps--secrets)
11. [Volumes & Persistent Storage](#volumes--persistent-storage)
12. [Networking & Ingress](#networking--ingress)
13. [Scaling & Autoscaling](#scaling--autoscaling)
14. [Rolling Updates & Rollbacks](#rolling-updates--rollbacks)
15. [Jobs & CronJobs](#jobs--cronjobs)
16. [RBAC & Security](#rbac--security)
17. [Helm - Package Manager](#helm---package-manager)
18. [Monitoring & Debugging](#monitoring--debugging)

---

## Introducing Kubernetes

Kubernetes (K8s) is an open-source container orchestration platform that automates deploying, scaling, and managing containerized applications. While Docker runs containers on a single machine, Kubernetes coordinates containers across a cluster of machines, handling failover, load balancing, and rolling updates automatically.

Kubernetes lets you run containers at scale across multiple nodes, self-heal by restarting crashed containers and replacing failed nodes, balance traffic across container replicas, roll out updates with zero downtime, manage configuration and secrets securely, and abstract away the underlying infrastructure so your app runs the same way on any cloud or on-premise cluster.

Core concepts of Kubernetes are:

- **Cluster**: A set of machines (nodes) that run containerized applications managed by Kubernetes.
- **Node**: A single machine (physical or virtual) in the cluster. Runs one or more Pods.
- **Pod**: The smallest deployable unit. A wrapper around one or more containers that share networking and storage.
- **Deployment**: Manages a set of identical Pods, handling scaling and rolling updates.
- **Service**: A stable network endpoint that routes traffic to a set of Pods.
- **Namespace**: A virtual partition within a cluster for organizing resources.
- **ConfigMap / Secret**: External configuration and sensitive data stored outside your container images.
- **Volume**: Storage that persists beyond the lifetime of a single container.

---

## Architecture & Components

A Kubernetes cluster consists of a Control Plane (master) and one or more Worker Nodes.

### Control Plane Components

| Component                  | Purpose                                                                             |
| -------------------------- | ----------------------------------------------------------------------------------- |
| `kube-apiserver`           | Front door to the cluster. All kubectl commands go through this REST API            |
| `etcd`                     | Distributed key-value store that holds all cluster state and configuration          |
| `kube-scheduler`           | Assigns newly created Pods to nodes based on resource requirements and constraints  |
| `kube-controller-manager`  | Runs control loops that watch cluster state and make changes to reach desired state |
| `cloud-controller-manager` | Integrates with cloud provider APIs (load balancers, volumes, routes)               |

### Worker Node Components

| Component           | Purpose                                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| `kubelet`           | Agent on each node that ensures containers in Pods are running and healthy                                 |
| `kube-proxy`        | Maintains network rules on each node. Handles routing traffic to the correct Pod                           |
| `Container Runtime` | Software that actually runs containers (containerd, CRI-O). Docker is no longer the default since K8s 1.24 |

You declare the desired state (e.g., "run 3 replicas of my app") in a YAML manifest. The API server stores this in etcd. Controllers continuously compare the desired state with the actual state and make adjustments. The scheduler places Pods on nodes. The kubelet on each node pulls images and starts containers.

---

## Installation & Setup

### Local Development with Minikube

Minikube creates a single-node Kubernetes cluster on your local machine, ideal for learning and development.

**macOS**:

```bash
brew install minikube
minikube start
```

**Linux**:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start
```

**Windows**: Use `winget install Kubernetes.minikube`.

`minikube start` creates a VM or container, installs Kubernetes inside it, and configures `kubectl` to connect to it.

### Install kubectl

`kubectl` is the command-line tool used to interact with any Kubernetes cluster.

**macOS**:

```bash
brew install kubectl
```

**Linux**:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl
```

### Verify Installation

```bash
kubectl version --client                # Show kubectl version
minikube status                         # Show cluster status
kubectl cluster-info                    # Show cluster endpoint info
kubectl get nodes                       # List all nodes in the cluster
```

`kubectl get nodes` should show one node with status `Ready`.

### Other Local Options

| Tool             | Description                                             |
| ---------------- | ------------------------------------------------------- |
| `kind`           | Kubernetes in Docker. Runs cluster nodes as containers  |
| `k3s`            | Lightweight K8s distribution by Rancher, great for edge |
| `Docker Desktop` | Built-in single-node K8s cluster (enable in settings)   |

---

## kubectl Basics

`kubectl` is the primary CLI for Kubernetes. Almost every operation follows the pattern: `kubectl <verb> <resource> [name] [flags]`.

### Common Verbs

| Verb           | Description                              |
| -------------- | ---------------------------------------- |
| `get`          | List resources                           |
| `describe`     | Show detailed info about a resource      |
| `create`       | Create a resource from a file or command |
| `apply`        | Create or update a resource from a file  |
| `delete`       | Remove a resource                        |
| `edit`         | Open resource YAML in your editor        |
| `logs`         | View container logs                      |
| `exec`         | Run a command inside a container         |
| `port-forward` | Forward a local port to a Pod            |

### Output Formats

```bash
kubectl get pods                          # Default table output
kubectl get pods -o wide                  # Extra columns (node, IP)
kubectl get pods -o yaml                  # Full YAML output
kubectl get pods -o json                  # Full JSON output
kubectl get pods -o name                  # Just resource names
```

`-o wide` adds columns like node name and pod IP. `-o yaml` and `-o json` output the complete resource definition, useful for debugging or saving as a file.

### Dry Run & Generating YAML

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
```

`--dry-run=client` validates the command and generates the resource definition without actually creating it. Pipe to a file to create YAML templates quickly instead of writing them by hand.

### Context & Configuration

```bash
kubectl config get-contexts              # List all contexts
kubectl config current-context           # Show active context
kubectl config use-context minikube      # Switch context
```

A context combines a cluster, user, and namespace. If you work with multiple clusters (dev, staging, production), contexts let you switch between them.

---

## Pods

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network namespace (same IP, same localhost) and storage volumes.

### Creating Pods Imperatively

```bash
kubectl run nginx --image=nginx          # Create a pod named "nginx"
kubectl run nginx --image=nginx --port=80
```

`kubectl run` creates a single Pod. `--port=80` sets the container port metadata but does not expose it externally.

### FastAPI Docker Application for Minikube

This section demonstrates how to containerize a FastAPI application and build it within Minikube's Docker daemon.

#### 1. Dockerfile

This `Dockerfile` uses a lightweight Python base image, installs the necessary dependencies, and sets the startup command to run the FastAPI server.

```Dockerfile
FROM python:3.14-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["fastapi", "run", "main.py", "--port", "8000"]
```

#### 2. Application Code (`main.py`)

A simple FastAPI application that returns a JSON message on the root endpoint.

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get('/')
def home():
    return {"message": "Hello from Docker"}
```

#### 3. Dependencies (`requirements.txt`)

We require the `fastapi` framework along with standard dependencies (which include a production ASGI server).

```txt
fastapi[standard]
```

#### 4. Building the Image in Minikube

To use the local image directly in Minikube without needing an external container registry, you can evaluate the `minikube docker-env` command. This points your local Docker client to Minikube's internal Docker daemon.

```bash
# build inside minikube's docker daemon
eval $(minikube docker-env)
docker build -t fastapi-app:latest .
```

### Pod YAML Manifest

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fastapi-pod
  labels:
    app: fastapi
spec:
  containers:
    - name: api
      image: fastapi-app:latest
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8000
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

`apiVersion` and `kind` tell Kubernetes what resource to create. `metadata.name` is the unique name of the Pod. `labels` are key-value pairs used for selection and grouping. `spec.containers` lists the containers inside the Pod. `containerPort` documents which port the container listens on. `resources.requests` is the minimum guaranteed resources; the scheduler uses this to place the Pod. `resources.limits` is the maximum; the container is throttled (CPU) or killed (memory) if it exceeds this. `250m` means 250 millicores (0.25 CPU). `64Mi` means 64 mebibytes of memory. `imagePullPolicy: IfNotPresent` means the image will be pulled only if it is not already present on the node.

```bash
kubectl apply -f pod.yaml                # Create or update the pod
```

`apply` is declarative: it creates the resource if it does not exist or updates it if it does.

### Forwarding ports

```bash
kubectl port-forward pod/fastapi-pod 8000:8000
```

This command forwards traffic from your local machine's port 8000 to the pod's port 8000.

Now, you can access the application at `http://localhost:8000`.

### Managing Pods

```bash
kubectl get pods                         # List pods in default namespace
kubectl get pods -A                      # List pods in all namespaces
kubectl describe pod fastapi-pod         # Detailed pod info + events
kubectl logs fastapi-pod                 # View container logs
kubectl logs -f fastapi-pod              # Follow logs in real time
kubectl exec -it fastapi-pod -- bash     # Open shell inside the pod
kubectl delete pod fastapi-pod           # Delete the pod
kubectl delete -f pod.yaml              # Delete resource defined in file
```

`-A` (all namespaces) shows Pods across the entire cluster. `describe` shows events, conditions, and configuration, essential for debugging. `--` separates kubectl arguments from the command to run inside the container.

### Multi-Container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: app
      image: fastapi-app:latest
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8000
    - name: sidecar
      image: busybox
      command:
        ["sh", "-c", "while true; do echo 'health check'; sleep 30; done"]
```

Multiple containers in a Pod share the same network (they communicate over `localhost`) and can share volumes. Common patterns include sidecar containers for logging, proxies, or monitoring.

```bash
kubectl apply -f multi-container-pod.yaml
kubectl port-forward pod/multi-container-pod 8000:8000
kubectl logs multi-container-pod -c sidecar -f
kubectl logs multi-container-pod -c app -f
```

---

## ReplicaSets

A ReplicaSet ensures a specified number of identical Pod replicas are running at all times. If a Pod crashes or is deleted, the ReplicaSet creates a new one.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: fastapi-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
        - name: api
          image: fastapi-app:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
```

`replicas: 3` means the ReplicaSet will ensure exactly 3 Pods are always running. `selector.matchLabels` tells the ReplicaSet which Pods it manages. `template` is the Pod specification used to create new Pods. The labels in `template.metadata.labels` must match `selector.matchLabels`.

### Forwarding ports

```bash
kubectl port-forward rs/fastapi-rs 8000:8000
```

This command forwards traffic from your local machine's port 8000 to the replicaset's port 8000.

```bash
kubectl apply -f replicaset.yaml
kubectl get rs                           # List ReplicaSets (rs is shorthand)
kubectl describe rs fastapi-rs
kubectl delete rs fastapi-rs

kubectl get pods
kubectl delete pod fastapi-rs-xxxxx
kubectl get pods      # still 3 pods (new created by rs)

```

In practice, you rarely create ReplicaSets directly. Deployments manage ReplicaSets for you and add rolling updates and rollback capabilities.

---

## Deployments

A Deployment is the recommended way to run stateless applications. It manages ReplicaSets, which in turn manage Pods. Deployments provide declarative updates, rolling updates, and rollbacks.

### Deployment YAML

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
        - name: api
          image: fastapi-app:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```

The structure is nearly identical to a ReplicaSet but `kind` is `Deployment`. The Deployment creates and manages a ReplicaSet behind the scenes.

### Forwarding ports

```bash
kubectl port-forward deploy/fastapi-deploy 8000:8000
```

This command forwards traffic from your local machine's port 8000 to the deployment's port 8000.

### Imperative Deployment

```bash
kubectl create deployment fastapi-deploy --image=yourusername/fastapi-app:1.0 --replicas=3
```

### Managing Deployments

```bash
kubectl apply -f deployment.yaml         # Create or update
kubectl get deployments                  # List deployments
kubectl get deploy                       # Shorthand
kubectl describe deploy fastapi-deploy   # Detailed info + events
kubectl delete deploy fastapi-deploy     # Delete deployment + its pods
```

### Viewing the Hierarchy

```bash
kubectl get deploy                       # Deployment
kubectl get rs                           # ReplicaSet (created by Deployment)
kubectl get pods                         # Pods (created by ReplicaSet)
```

A Deployment creates a ReplicaSet, which creates Pods. The chain is Deployment → ReplicaSet → Pods.

---

## Services

Pods get new IP addresses every time they restart. A Service provides a stable endpoint (fixed IP and DNS name) that routes traffic to a set of Pods based on label selectors.

### Service Types

| Type           | Description                                                                               |
| -------------- | ----------------------------------------------------------------------------------------- |
| `ClusterIP`    | Default. Exposes the Service on an internal cluster IP. Only reachable within the cluster |
| `NodePort`     | Exposes the Service on each Node's IP at a static port (30000–32767)                      |
| `LoadBalancer` | Provisions a cloud provider load balancer that routes external traffic to the Service     |
| `ExternalName` | Maps the Service to a DNS name (CNAME record). No proxying                                |

### ClusterIP Service

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  type: ClusterIP
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

`selector.app: fastapi` matches all Pods with label `app: fastapi`. `port: 80` is the port the Service listens on within the cluster. `targetPort: 8000` is the port on the Pod where traffic is forwarded. Other Pods in the cluster can reach this Service at `fastapi-service:80`.

### NodePort Service

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-nodeport
spec:
  type: NodePort
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
      nodePort: 30080
```

`nodePort: 30080` exposes the Service on port 30080 on every node. Access the app at `<any-node-ip>:30080`. If you omit `nodePort`, Kubernetes assigns a random port in the 30000–32767 range.

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-lb
spec:
  type: LoadBalancer
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

On cloud providers (AWS, GCP, Azure), this automatically provisions an external load balancer. On Minikube, use `minikube service fastapi-lb` to open it.

### Managing Services

```bash
kubectl apply -f service-clusterip.yaml
kubectl get svc                          # List services (svc is shorthand)
kubectl describe svc fastapi-service
kubectl delete svc fastapi-service
```

### Quick Expose

```bash
kubectl expose deployment fastapi-deploy --type=NodePort --port=80 --target-port=8000
```

`expose` creates a Service for an existing Deployment without writing YAML.

### Accessing on Minikube

```bash
minikube service fastapi-nodeport        # Opens the service URL in browser
minikube service fastapi-nodeport --url  # Just print the URL
```

---

## Namespaces

Namespaces are virtual clusters within a physical cluster. They provide isolation, resource quotas, and organization for teams or environments.

### Default Namespaces

| Namespace         | Purpose                                              |
| ----------------- | ---------------------------------------------------- |
| `default`         | Where resources go if no namespace is specified      |
| `kube-system`     | Kubernetes system components (API server, scheduler) |
| `kube-public`     | Publicly readable data, like cluster info            |
| `kube-node-lease` | Node heartbeat data for node health detection        |

### Working with Namespaces

```bash
kubectl get namespaces                   # List all namespaces
kubectl get ns                           # Shorthand
kubectl create namespace dev             # Create a namespace
kubectl get pods -n dev                  # List pods in "dev" namespace
kubectl get all -n dev                   # List all resources in "dev"
kubectl delete namespace dev             # Delete namespace and all its resources
```

`-n dev` targets the `dev` namespace. Without `-n`, kubectl uses the `default` namespace.

### Setting Default Namespace

```bash
kubectl config set-context --current --namespace=dev
```

This modifies your kubeconfig so all subsequent commands target the `dev` namespace without needing `-n`.

### Namespace in YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fastapi-pod
  namespace: dev
  labels:
    app: fastapi
spec:
  containers:
    - name: api
      image: yourusername/fastapi-app:1.0
```

`namespace: dev` creates the resource in the `dev` namespace.

### Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```

ResourceQuotas limit the total resources a namespace can consume, preventing a single team or environment from monopolizing cluster resources.

---

## ConfigMaps & Secrets

ConfigMaps store non-sensitive configuration data. Secrets store sensitive data (passwords, tokens, keys). Both decouple configuration from container images.

### ConfigMaps

```bash
# Create from literal values
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info

# Create from a file
kubectl create configmap app-config --from-file=config.properties

# Create from an env file
kubectl create configmap app-config --from-env-file=.env
```

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  DATABASE_HOST: db-service
```

### Using ConfigMaps in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fastapi-pod
spec:
  containers:
    - name: api
      image: yourusername/fastapi-app:1.0
      # Option 1: Inject all keys as environment variables
      envFrom:
        - configMapRef:
            name: app-config
      # Option 2: Inject specific keys
      env:
        - name: APP_ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
```

`envFrom` injects all keys from the ConfigMap as environment variables. `valueFrom.configMapKeyRef` injects a single key, optionally renaming it.

### Secrets

```bash
kubectl create secret generic db-creds --from-literal=DB_USER=admin --from-literal=DB_PASS=supersecret
kubectl get secrets
kubectl describe secret db-creds
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  DB_USER: YWRtaW4= # base64 encoded "admin"
  DB_PASS: c3VwZXJzZWNyZXQ= # base64 encoded "supersecret"
```

Secret values in YAML must be base64-encoded. Encode with `echo -n 'admin' | base64`. The `-n` flag prevents a trailing newline. Kubernetes stores Secrets encoded, not encrypted by default. Enable encryption at rest for production clusters.

### Using Secrets in Pods

```yaml
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: DB_USER
  - name: DB_PASS
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: DB_PASS
```

### Mounting as Files

```yaml
spec:
  containers:
    - name: api
      image: yourusername/fastapi-app:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

Each key in the ConfigMap becomes a file in `/etc/config/`. The filename is the key and the content is the value. Secrets can be mounted the same way.

---

## Volumes & Persistent Storage

Container filesystems are ephemeral. Kubernetes Volumes provide persistent storage that survives container restarts and can be shared between containers in a Pod.

### Volume Types

| Type                    | Description                                                                   |
| ----------------------- | ----------------------------------------------------------------------------- |
| `emptyDir`              | Temporary storage that exists while the Pod runs. Deleted when Pod is removed |
| `hostPath`              | Mounts a file or directory from the host node's filesystem                    |
| `persistentVolumeClaim` | Claims storage from a PersistentVolume. The standard approach                 |
| `configMap` / `secret`  | Mount ConfigMaps or Secrets as files                                          |
| `nfs`                   | Network File System mount                                                     |
| Cloud volumes           | `awsElasticBlockStore`, `gcePersistentDisk`, `azureDisk`                      |

### emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-pod
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /data/message && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh", "-c", "cat /data/message && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}
```

`emptyDir` creates an empty directory when the Pod starts. Both containers can read and write to it. The data is lost when the Pod is deleted.

### PersistentVolume (PV) & PersistentVolumeClaim (PVC)

PersistentVolumes are cluster-level storage resources provisioned by an admin or dynamically by a StorageClass. PersistentVolumeClaims are requests for storage by users.

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
```

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

`capacity.storage: 1Gi` defines the volume size. `ReadWriteOnce` means only one node can mount it for reading and writing at a time. Other access modes are `ReadOnlyMany` (multiple nodes read-only) and `ReadWriteMany` (multiple nodes read-write). `hostPath` is for development only; in production, use cloud-backed or network storage.

### Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fastapi-pod
spec:
  containers:
    - name: api
      image: yourusername/fastapi-app:1.0
      volumeMounts:
        - name: app-storage
          mountPath: /app/data
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: app-pvc
```

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl get pv                           # List PersistentVolumes
kubectl get pvc                          # List PersistentVolumeClaims
```

The PVC binds to a PV that satisfies its requirements. Once bound, the Pod can mount the PVC as a volume.

### StorageClasses (Dynamic Provisioning)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
```

```yaml
# pvc with StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 5Gi
```

`StorageClass` enables dynamic provisioning. When a PVC references a StorageClass, Kubernetes automatically creates a PV from the cloud provider. `reclaimPolicy: Delete` deletes the underlying storage when the PVC is deleted. Use `Retain` to keep the data.

---

## Networking & Ingress

### Kubernetes Networking Model

Every Pod gets its own unique IP address. Pods can communicate with all other Pods across nodes without NAT. Services provide stable endpoints for Pod groups.

### DNS

Kubernetes runs a DNS service (CoreDNS) inside the cluster. Services are reachable by name:

```
<service-name>.<namespace>.svc.cluster.local
```

For example, a Service named `db-service` in the `default` namespace is reachable at `db-service.default.svc.cluster.local`, or simply `db-service` from the same namespace.

### Port Forwarding (Development)

```bash
kubectl port-forward pod/fastapi-pod 8080:8000       # Forward local 8080 to pod 8000
kubectl port-forward svc/fastapi-service 8080:80     # Forward local 8080 to service 80
kubectl port-forward deploy/fastapi-deploy 8080:8000 # Forward to deployment
```

`port-forward` creates a tunnel from your local machine to a resource inside the cluster. Useful for debugging without exposing services externally.

### Ingress

An Ingress exposes HTTP and HTTPS routes from outside the cluster to Services inside the cluster. It provides URL-based routing, SSL termination, and name-based virtual hosting.

**Install an Ingress Controller** (required before Ingress resources work):

```bash
# Minikube
minikube addons enable ingress

# Or install NGINX Ingress Controller via Helm
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: fastapi-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.local
      secretName: tls-secret
```

`rules` define how incoming requests are routed based on hostname and path. `pathType: Prefix` matches URLs starting with the specified path. `backend` points to the Service and port to forward traffic to. `tls` enables HTTPS using a certificate stored in a Kubernetes Secret. `annotations` configure behavior specific to the Ingress controller.

```bash
kubectl apply -f ingress.yaml
kubectl get ingress                      # List Ingress resources
kubectl describe ingress app-ingress
```

For Minikube, add the Ingress IP to `/etc/hosts`:

```bash
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts
```

---

## Scaling & Autoscaling

### Manual Scaling

```bash
kubectl scale deployment fastapi-deploy --replicas=5     # Scale to 5 pods
kubectl scale deployment fastapi-deploy --replicas=1     # Scale down to 1
```

`scale` changes the number of Pod replicas immediately. Kubernetes creates or terminates Pods to match the desired count.

### Horizontal Pod Autoscaler (HPA)

HPA automatically adjusts the number of Pod replicas based on observed CPU utilization or custom metrics.

```bash
# Requires metrics-server installed
minikube addons enable metrics-server    # Enable on Minikube

kubectl autoscale deployment fastapi-deploy --min=2 --max=10 --cpu-percent=50
```

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

`minReplicas` and `maxReplicas` set the scaling bounds. `averageUtilization: 50` means if average CPU across all Pods exceeds 50%, HPA scales up; if below, it scales down. Pods must have `resources.requests.cpu` set for HPA to work.

```bash
kubectl get hpa                          # Check HPA status
kubectl describe hpa fastapi-hpa
kubectl delete hpa fastapi-hpa
```

### Vertical Pod Autoscaler (VPA)

VPA adjusts the CPU and memory requests/limits of containers automatically. It requires the VPA addon and is complementary to HPA (do not use both for the same metric).

---

## Rolling Updates & Rollbacks

Deployments perform rolling updates by default, replacing Pods incrementally to maintain availability.

### Updating an Image

```bash
kubectl set image deployment/fastapi-deploy api=yourusername/fastapi-app:2.0
```

`set image` updates the container image. Kubernetes creates new Pods with the updated image and terminates old Pods gradually.

### Update Strategy

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

`maxSurge: 1` allows 1 extra Pod above the desired count during the update. `maxUnavailable: 0` ensures no Pods are taken down before new ones are ready. This guarantees zero-downtime deployments.

### Monitoring Rollouts

```bash
kubectl rollout status deployment/fastapi-deploy    # Watch rollout progress
kubectl rollout history deployment/fastapi-deploy   # View rollout history
```

### Rollbacks

```bash
kubectl rollout undo deployment/fastapi-deploy              # Rollback to previous version
kubectl rollout undo deployment/fastapi-deploy --to-revision=2  # Rollback to specific revision
```

`undo` reverts the Deployment to a previous ReplicaSet. Kubernetes keeps a history of revisions (default 10) for rollbacks.

---

## Jobs & CronJobs

### Jobs

A Job creates one or more Pods and ensures they complete successfully. Used for batch processing, database migrations, or one-time tasks.

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  completions: 1
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: migrate
          image: yourusername/fastapi-app:1.0
          command: ["python", "migrate.py"]
      restartPolicy: Never
```

`completions: 1` means the Job is complete after 1 successful Pod. `backoffLimit: 3` retries up to 3 times on failure. `restartPolicy: Never` means failed Pods are not restarted; instead, a new Pod is created. Jobs require `restartPolicy` to be `Never` or `OnFailure`.

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/db-migrate
kubectl delete job db-migrate
```

### CronJobs

CronJobs run Jobs on a schedule, similar to Unix cron.

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: yourusername/backup-tool:1.0
              command: ["./backup.sh"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

`schedule` uses standard cron syntax (minute hour day-of-month month day-of-week). `"0 2 * * *"` runs daily at 2:00 AM. `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` control how many completed Job records are kept.

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get cj                           # Shorthand
kubectl describe cronjob daily-backup
kubectl delete cronjob daily-backup
```

---

## RBAC & Security

Role-Based Access Control (RBAC) restricts who can perform what actions on which resources in the cluster.

### Core Concepts

| Resource             | Scope     | Description                                      |
| -------------------- | --------- | ------------------------------------------------ |
| `Role`               | Namespace | Defines permissions within a single namespace    |
| `ClusterRole`        | Cluster   | Defines permissions cluster-wide                 |
| `RoleBinding`        | Namespace | Binds a Role to users/groups within a namespace  |
| `ClusterRoleBinding` | Cluster   | Binds a ClusterRole to users/groups cluster-wide |
| `ServiceAccount`     | Namespace | Identity for processes running in Pods           |

### Creating a Role & RoleBinding

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

`apiGroups: [""]` means the core API group. `resources: ["pods"]` targets Pod resources. `verbs` specify the allowed actions. `subjects` defines who gets the permissions. `roleRef` points to the Role being granted.

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
kubectl get roles -n dev
kubectl get rolebindings -n dev
```

### ServiceAccounts

```bash
kubectl create serviceaccount app-sa -n dev
```

```yaml
spec:
  serviceAccountName: app-sa
  containers:
    - name: api
      image: yourusername/fastapi-app:1.0
```

`serviceAccountName` assigns a ServiceAccount to a Pod, giving it specific permissions within the cluster.

### Security Best Practices

- Run containers as non-root users using `securityContext.runAsNonRoot: true`
- Set `readOnlyRootFilesystem: true` to prevent writes to the container filesystem
- Drop all Linux capabilities with `securityContext.capabilities.drop: ["ALL"]`
- Use Network Policies to restrict Pod-to-Pod communication
- Scan container images for vulnerabilities before deploying

---

## Helm - Package Manager

Helm is the package manager for Kubernetes. It bundles Kubernetes YAML manifests into reusable packages called **charts**.

### Install Helm

**macOS**:

```bash
brew install helm
```

**Linux**:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Key Concepts

| Term         | Description                                               |
| ------------ | --------------------------------------------------------- |
| `Chart`      | A package of Kubernetes resource templates                |
| `Release`    | A deployed instance of a chart in your cluster            |
| `Repository` | A collection of charts hosted for sharing                 |
| `Values`     | Configuration that customizes a chart during installation |

### Common Commands

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami    # Add a chart repository
helm repo update                         # Update repository index
helm search repo nginx                   # Search for charts
helm search hub postgres                 # Search Artifact Hub

helm install my-nginx bitnami/nginx      # Install a chart as "my-nginx"
helm install my-nginx bitnami/nginx -f values.yaml  # Install with custom values
helm install my-nginx bitnami/nginx --set replicaCount=3    # Override a value

helm list                                # List installed releases
helm status my-nginx                     # Show release status
helm upgrade my-nginx bitnami/nginx --set replicaCount=5    # Upgrade a release
helm rollback my-nginx 1                 # Rollback to revision 1
helm uninstall my-nginx                  # Remove a release
```

`helm install` deploys the chart to your cluster and creates a release. `-f values.yaml` provides custom configuration. `--set` overrides individual values. `helm upgrade` applies changes to an existing release. `helm rollback` reverts to a previous release version.

### Creating a Chart

```bash
helm create my-chart                     # Scaffold a new chart
```

This creates a directory with the following structure:

```
my-chart/
├── Chart.yaml                           # Chart metadata (name, version)
├── values.yaml                          # Default configuration values
├── templates/                           # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl                     # Template helper functions
└── charts/                              # Dependency charts
```

`Chart.yaml` defines the chart's name, version, and description. `values.yaml` contains defaults that users can override. Templates use Go templating syntax (`{{ .Values.replicaCount }}`) to inject values.

---

## Monitoring & Debugging

### Viewing Resource Status

```bash
kubectl get all                          # List all resources in current namespace
kubectl get all -A                       # List all resources across all namespaces
kubectl get events                       # Show cluster events
kubectl get events --sort-by=.metadata.creationTimestamp   # Events sorted by time
```

`get all` lists Deployments, ReplicaSets, Pods, Services, and more in one command. `events` show what is happening in the cluster: scheduling, pulling images, errors.

### Debugging Pods

```bash
kubectl describe pod <pod-name>          # Detailed info + events
kubectl logs <pod-name>                  # Container logs
kubectl logs <pod-name> -c <container>   # Logs from specific container
kubectl logs -f <pod-name>              # Follow logs
kubectl logs --previous <pod-name>       # Logs from a crashed container
kubectl exec -it <pod-name> -- sh        # Shell into the container
```

`describe` is the first tool for debugging. Check the `Events` section at the bottom for errors like image pull failures, insufficient resources, or failed health checks. `--previous` retrieves logs from the last terminated container, essential for debugging crash loops.

### Resource Usage

```bash
kubectl top nodes                        # CPU/memory usage per node
kubectl top pods                         # CPU/memory usage per pod
kubectl top pods -A                      # All namespaces
```

`top` requires the metrics-server to be running. On Minikube: `minikube addons enable metrics-server`.

### Common Pod Issues

| Status             | Meaning                                 | Debug Steps                               |
| ------------------ | --------------------------------------- | ----------------------------------------- |
| `Pending`          | Pod cannot be scheduled                 | Check `describe` for resource issues      |
| `CrashLoopBackOff` | Container keeps crashing and restarting | Check `logs --previous` for errors        |
| `ImagePullBackOff` | Cannot pull the container image         | Verify image name, tag, and registry auth |
| `ErrImagePull`     | Failed to pull image                    | Check image exists, credentials correct   |
| `OOMKilled`        | Container exceeded memory limit         | Increase `resources.limits.memory`        |
| `Evicted`          | Node ran out of resources               | Check node capacity with `top nodes`      |

### Health Checks (Probes)

```yaml
spec:
  containers:
    - name: api
      image: yourusername/fastapi-app:1.0
      livenessProbe:
        httpGet:
          path: /
          port: 8000
        initialDelaySeconds: 10
        periodSeconds: 15
      readinessProbe:
        httpGet:
          path: /
          port: 8000
        initialDelaySeconds: 5
        periodSeconds: 10
      startupProbe:
        httpGet:
          path: /
          port: 8000
        failureThreshold: 30
        periodSeconds: 5
```

`livenessProbe` checks if the container is still running. If it fails, Kubernetes restarts the container. `readinessProbe` checks if the container is ready to receive traffic. If it fails, the Pod is removed from Service endpoints. `startupProbe` disables liveness and readiness checks until the container starts successfully, useful for slow-starting applications. `initialDelaySeconds` waits before the first probe. `periodSeconds` is the interval between probes. `failureThreshold` is how many consecutive failures before action is taken.

### Useful Debugging Commands

```bash
kubectl get pod <pod-name> -o yaml       # Full pod definition
kubectl get endpoints <service-name>     # Check which pods a service targets
kubectl run debug --image=busybox -it --rm -- sh   # Temporary debug pod
kubectl auth can-i create pods           # Check RBAC permissions
kubectl auth can-i create pods --as jane # Check permissions for another user
kubectl api-resources                    # List all resource types
kubectl explain deployment.spec          # Documentation for a resource field
```

`--rm` deletes the debug pod when you exit. `kubectl explain` provides built-in documentation for any resource field, useful for learning YAML structure without leaving the terminal.
