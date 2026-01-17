# ☸️ Chapter 06: Pods Deep Dive

> Understanding the fundamental building block of Kubernetes - where your containers actually run.

---

## 📚 Table of Contents

1. [What is a Pod?](#-what-is-a-pod)
2. [Pod Anatomy](#-pod-anatomy)
3. [Pod Lifecycle](#-pod-lifecycle)
4. [Creating Pods](#-creating-pods)
5. [Multi-Container Pods](#-multi-container-pods)
6. [Init Containers](#-init-containers)
7. [Pod Networking](#-pod-networking)
8. [Resource Management](#-resource-management)
9. [Health Probes](#-health-probes)
10. [Environment Variables](#-environment-variables)
11. [Volumes in Pods](#-volumes-in-pods)
12. [Pod Security](#-pod-security)
13. [Pod Scheduling](#-pod-scheduling)
14. [Troubleshooting Pods](#-troubleshooting-pods)
15. [CKA Exam Tips](#-cka-exam-tips)
16. [Summary](#-summary)

---

## 📖 What is a Pod?

### Definition

> **Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a running process and can contain one or more containers that share storage and network.

### Key Concepts

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              WHAT IS A POD?                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  A Pod is NOT a container. A Pod CONTAINS containers.                               │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │                               POD                                              ││
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ ││
│  │  │  Shared:                                                                 │ ││
│  │  │  • Network namespace (same IP, localhost works)                          │ ││
│  │  │  • IPC namespace (can use shared memory)                                 │ ││
│  │  │  • Volumes (shared storage)                                              │ ││
│  │  └──────────────────────────────────────────────────────────────────────────┘ ││
│  │                                                                                ││
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              ││
│  │  │   Container 1   │  │   Container 2   │  │   Container 3   │              ││
│  │  │   (main app)    │  │   (sidecar)     │  │   (optional)    │              ││
│  │  │                 │  │                 │  │                 │              ││
│  │  │  localhost:8080 │  │  localhost:9090 │  │  localhost:9100 │              ││
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘              ││
│  │                                                                                ││
│  │  Pod IP: 10.244.1.5  ◄─── All containers share this IP                       ││
│  │                                                                                ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  Key Points:                                                                        │
│  • Containers in a pod are always co-located on the same node                      │
│  • Containers in a pod can communicate via localhost                               │
│  • Pods are ephemeral - they can be killed and recreated anytime                   │
│  • Each pod gets a unique IP address                                               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Pod vs Container vs Docker

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DOCKER VS KUBERNETES MAPPING                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Docker                              Kubernetes                                      │
│  ──────                              ──────────                                      │
│  docker run nginx                    kubectl run nginx --image=nginx (creates Pod)  │
│                                                                                      │
│  Container (standalone)              Pod (wraps container(s))                       │
│  ┌─────────────────┐                 ┌──────────────────────────────────┐           │
│  │   Container     │                 │  Pod                             │           │
│  │   ┌─────────┐   │                 │  ┌─────────────────────────────┐ │           │
│  │   │  nginx  │   │                 │  │  Container                  │ │           │
│  │   └─────────┘   │                 │  │  ┌─────────────────────┐    │ │           │
│  └─────────────────┘                 │  │  │       nginx        │    │ │           │
│                                      │  │  └─────────────────────┘    │ │           │
│                                      │  └─────────────────────────────┘ │           │
│                                      └──────────────────────────────────┘           │
│                                                                                      │
│  Why the extra layer?                                                               │
│  • Pods can contain multiple related containers                                     │
│  • Pods provide shared networking and storage                                       │
│  • Pods are the unit of scheduling (not containers)                                │
│  • Pods enable the sidecar pattern                                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔬 Pod Anatomy

### Complete Pod Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: complete-pod-example
  namespace: default
  labels:
    app: myapp
    tier: frontend
  annotations:
    description: "A complete pod example"
spec:
  # Node selection
  nodeName: worker-1                    # Specific node (rarely used)
  nodeSelector:                         # Label-based selection
    disktype: ssd
  
  # Service account
  serviceAccountName: my-service-account
  
  # Security
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  
  # Init containers (run first, in order)
  initContainers:
  - name: init-db
    image: busybox:1.28
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  
  # Main containers
  containers:
  - name: main-app
    image: nginx:1.19
    
    # Command and args
    command: ["nginx"]
    args: ["-g", "daemon off;"]
    
    # Ports
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    
    # Environment variables
    env:
    - name: NODE_ENV
      value: "production"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # Environment from ConfigMap/Secret
    envFrom:
    - configMapRef:
        name: app-config
    
    # Resource requests and limits
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    
    # Volume mounts
    volumeMounts:
    - name: data-volume
      mountPath: /data
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
    
    # Health probes
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
    
    startupProbe:
      httpGet:
        path: /startup
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    
    # Container security
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
  
  # Sidecar container
  - name: sidecar-logger
    image: busybox:1.28
    command: ['sh', '-c', 'tail -f /var/log/app.log']
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  
  # Volumes
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc
  - name: config-volume
    configMap:
      name: app-config
  - name: log-volume
    emptyDir: {}
  
  # Restart policy
  restartPolicy: Always          # Always, OnFailure, Never
  
  # Termination
  terminationGracePeriodSeconds: 30
  
  # DNS
  dnsPolicy: ClusterFirst
  
  # Tolerations (for taints)
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
  
  # Affinity
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
```

---

## 🔄 Pod Lifecycle

### Pod Phases

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              POD PHASES                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Phase      │ Description                                                           │
│  ───────────┼───────────────────────────────────────────────────────────────────────│
│  Pending    │ Pod accepted but not yet running                                      │
│             │ • Waiting for scheduling                                              │
│             │ • Downloading images                                                  │
│             │ • Waiting for volumes                                                 │
│  ───────────┼───────────────────────────────────────────────────────────────────────│
│  Running    │ Pod bound to node, all containers created                            │
│             │ • At least one container running or starting                         │
│  ───────────┼───────────────────────────────────────────────────────────────────────│
│  Succeeded  │ All containers terminated successfully                               │
│             │ • Exit code 0                                                         │
│             │ • Won't be restarted                                                  │
│  ───────────┼───────────────────────────────────────────────────────────────────────│
│  Failed     │ All containers terminated, at least one failed                       │
│             │ • Exit code non-zero or killed by system                             │
│  ───────────┼───────────────────────────────────────────────────────────────────────│
│  Unknown    │ Pod state cannot be determined                                        │
│             │ • Usually communication error with node                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Container States

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           CONTAINER STATES                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  State       │ Description                                                          │
│  ────────────┼──────────────────────────────────────────────────────────────────────│
│  Waiting     │ Container not yet running                                            │
│              │ • Pulling image                                                      │
│              │ • Waiting for secrets/configmaps                                    │
│              │ Reasons: ContainerCreating, CrashLoopBackOff, ImagePullBackOff     │
│  ────────────┼──────────────────────────────────────────────────────────────────────│
│  Running     │ Container executing without issues                                   │
│              │ • Started at specific time                                          │
│  ────────────┼──────────────────────────────────────────────────────────────────────│
│  Terminated  │ Container finished execution                                         │
│              │ • Either ran to completion or failed                                │
│              │ • Has exit code and reason                                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Lifecycle Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           POD LIFECYCLE FLOW                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   kubectl apply                                                                      │
│        │                                                                             │
│        ▼                                                                             │
│   ┌─────────┐    Schedule    ┌─────────────┐                                        │
│   │ Pending │───────────────>│  Scheduled  │                                        │
│   └─────────┘                └──────┬──────┘                                        │
│                                     │                                                │
│                                     ▼                                                │
│                           ┌─────────────────┐                                       │
│                           │  Init Containers │ (if any, run in order)              │
│                           └────────┬────────┘                                       │
│                                    │                                                 │
│                                    ▼                                                 │
│                           ┌─────────────────┐                                       │
│                           │   Pull Images   │                                       │
│                           └────────┬────────┘                                       │
│                                    │                                                 │
│                                    ▼                                                 │
│                           ┌─────────────────┐                                       │
│                           │ Create Containers│                                       │
│                           └────────┬────────┘                                       │
│                                    │                                                 │
│                                    ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────┐    │
│   │                           RUNNING                                          │    │
│   │                                                                            │    │
│   │  postStart hook ─> Main container ─> Probes ─> preStop hook              │    │
│   │                                                                            │    │
│   └─────────────────────────────────┬─────────────────────────────────────────┘    │
│                                     │                                                │
│                    ┌────────────────┴────────────────┐                              │
│                    │                                 │                               │
│                    ▼                                 ▼                               │
│              ┌───────────┐                    ┌──────────┐                          │
│              │ Succeeded │                    │  Failed  │                          │
│              │ (exit 0)  │                    │ (exit !0)│                          │
│              └───────────┘                    └──────────┘                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Creating Pods

### Imperative Creation

```bash
# Simple pod
kubectl run nginx --image=nginx

# With specific options
kubectl run nginx --image=nginx:1.19 --port=80

# With labels
kubectl run nginx --image=nginx --labels="app=nginx,tier=frontend"

# With environment variables
kubectl run nginx --image=nginx --env="NODE_ENV=production"

# Dry run to generate YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Run and delete immediately (for testing)
kubectl run test --image=busybox:1.28 --rm -it -- /bin/sh
```

### Declarative Creation

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.19
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
```

### Essential Pod Operations

```bash
# List pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get pods -w  # Watch

# Describe pod
kubectl describe pod nginx

# View logs
kubectl logs nginx
kubectl logs nginx -f  # Follow
kubectl logs nginx --previous  # Previous container

# Execute command
kubectl exec nginx -- ls /
kubectl exec -it nginx -- /bin/bash

# Port forward
kubectl port-forward pod/nginx 8080:80

# Delete pod
kubectl delete pod nginx
kubectl delete pod nginx --force --grace-period=0  # Force delete
```

---

## 🔗 Multi-Container Pods

### Patterns

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       MULTI-CONTAINER POD PATTERNS                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. SIDECAR PATTERN                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Main container + helper container                                                  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────┐                             │
│  │                      Pod                          │                             │
│  │  ┌─────────────────┐    ┌─────────────────┐      │                             │
│  │  │   Main App      │    │   Log Shipper   │      │  Sidecar sends logs        │
│  │  │   (nginx)       │───>│   (fluentd)     │───>  │  to central system         │
│  │  └─────────────────┘    └─────────────────┘      │                             │
│  │         │                       │                 │                             │
│  │         └───────────────────────┘                 │                             │
│  │              Shared volume: /var/log              │                             │
│  └───────────────────────────────────────────────────┘                             │
│                                                                                      │
│  2. AMBASSADOR PATTERN                                                              │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Proxy for external communication                                                   │
│                                                                                      │
│  ┌───────────────────────────────────────────────────┐                             │
│  │                      Pod                          │                             │
│  │  ┌─────────────────┐    ┌─────────────────┐      │                             │
│  │  │   Main App      │───>│   Ambassador    │───>  │  Redis Cluster             │
│  │  │                 │    │   (Proxy)       │      │                             │
│  │  │ connects to     │    │                 │      │                             │
│  │  │ localhost:6379  │    │ Routes to       │      │                             │
│  │  └─────────────────┘    │ correct shard   │      │                             │
│  │                         └─────────────────┘      │                             │
│  └───────────────────────────────────────────────────┘                             │
│                                                                                      │
│  3. ADAPTER PATTERN                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Transform output for consumption                                                   │
│                                                                                      │
│  ┌───────────────────────────────────────────────────┐                             │
│  │                      Pod                          │                             │
│  │  ┌─────────────────┐    ┌─────────────────┐      │                             │
│  │  │   Main App      │───>│   Adapter       │───>  │  Prometheus                │
│  │  │                 │    │   (Exporter)    │      │                             │
│  │  │ Custom metrics  │    │                 │      │                             │
│  │  │ format          │    │ Converts to     │      │                             │
│  │  └─────────────────┘    │ Prometheus fmt  │      │                             │
│  │                         └─────────────────┘      │                             │
│  └───────────────────────────────────────────────────┘                             │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Sidecar Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-example
spec:
  containers:
  # Main application
  - name: main-app
    image: nginx
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  # Sidecar: Log shipper
  - name: log-shipper
    image: busybox:1.28
    command: ['sh', '-c', 'tail -f /var/log/nginx/access.log']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  volumes:
  - name: shared-logs
    emptyDir: {}
```

### Accessing Specific Container

```bash
# Logs from specific container
kubectl logs sidecar-example -c main-app
kubectl logs sidecar-example -c log-shipper

# Exec into specific container
kubectl exec -it sidecar-example -c main-app -- /bin/bash
kubectl exec -it sidecar-example -c log-shipper -- /bin/sh
```

---

## 🚀 Init Containers

### What are Init Containers?

> **Init Containers** are specialized containers that run before app containers in a Pod. They run to completion, one at a time, in order.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           INIT CONTAINERS                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Pod Startup Order:                                                                 │
│                                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  Init 1     │───>│  Init 2     │───>│  Init 3     │───>│  App Containers     │  │
│  │  (complete) │    │  (complete) │    │  (complete) │    │  (start together)   │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────────────┘  │
│                                                                                      │
│  If Init 1 fails, Pod restarts. Init containers run again.                         │
│                                                                                      │
│  Use Cases:                                                                         │
│  • Wait for a service to be available                                              │
│  • Clone a git repo for the app                                                    │
│  • Generate config files                                                           │
│  • Run database migrations                                                         │
│  • Download dependencies                                                           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Init Container Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  # Wait for database to be ready
  - name: wait-for-db
    image: busybox:1.28
    command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting for db; sleep 2; done']
  
  # Download config from external source
  - name: download-config
    image: busybox:1.28
    command: ['sh', '-c', 'wget -O /config/app.conf http://config-server/app.conf']
    volumeMounts:
    - name: config
      mountPath: /config
  
  # Main application
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config
      mountPath: /etc/app
  
  volumes:
  - name: config
    emptyDir: {}
```

---

## 🌐 Pod Networking

### Networking Fundamentals

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           POD NETWORKING                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Key Principles:                                                                    │
│  1. Every pod gets its own unique IP address                                       │
│  2. Pods can communicate with any other pod without NAT                            │
│  3. Agents on a node can communicate with all pods on that node                    │
│                                                                                      │
│  Within a Pod:                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Pod (IP: 10.244.1.5)                                                        │  │
│  │                                                                              │  │
│  │  ┌──────────────────┐      ┌──────────────────┐                             │  │
│  │  │  Container A     │      │  Container B     │                             │  │
│  │  │  localhost:8080  │◄────►│  localhost:9090  │                             │  │
│  │  │                  │      │                  │  Containers communicate    │  │
│  │  └──────────────────┘      └──────────────────┘  via localhost             │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  Between Pods:                                                                      │
│  ┌─────────────────────────┐            ┌─────────────────────────┐               │
│  │  Pod A (10.244.1.5)     │            │  Pod B (10.244.2.10)    │               │
│  │  ┌───────────────────┐  │            │  ┌───────────────────┐  │               │
│  │  │  Container        │  │───────────►│  │  Container        │  │               │
│  │  │                   │  │  Direct IP │  │                   │  │               │
│  │  └───────────────────┘  │  to IP     │  └───────────────────┘  │               │
│  └─────────────────────────┘            └─────────────────────────┘               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Port Configuration

```yaml
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - name: http           # Named port (can be referenced)
      containerPort: 80    # Port the container listens on
      protocol: TCP        # TCP (default), UDP, or SCTP
    - name: https
      containerPort: 443
    - name: metrics
      containerPort: 9090
```

---

## 📊 Resource Management

### Requests and Limits

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       RESOURCE REQUESTS VS LIMITS                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  REQUESTS = What the container needs (guaranteed)                                   │
│  LIMITS   = Maximum the container can use                                           │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                              │  │
│  │   0                    Request              Limit                  Node Max  │  │
│  │   │─────────────────────│───────────────────│──────────────────────│        │  │
│  │                         │                   │                                │  │
│  │   [     Guaranteed     ][   Burstable      ][        Killed       ]         │  │
│  │                                                                              │  │
│  │   • Below request: Always available                                         │  │
│  │   • Request to limit: Available if resources free                          │  │
│  │   • Above limit: Throttled (CPU) or OOM killed (memory)                     │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  CPU:                                                                               │
│  • Measured in cores or millicores (m)                                             │
│  • 1 CPU = 1000m = 1 vCPU/core                                                     │
│  • 100m = 0.1 CPU                                                                  │
│  • CPU is compressible - throttled when exceeded                                   │
│                                                                                      │
│  Memory:                                                                            │
│  • Measured in bytes (Ki, Mi, Gi)                                                  │
│  • Memory is NOT compressible - OOM killed when exceeded                           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Resource Configuration

```yaml
spec:
  containers:
  - name: app
    image: myapp
    resources:
      requests:
        memory: "64Mi"     # 64 Mebibytes
        cpu: "250m"        # 0.25 CPU cores
      limits:
        memory: "128Mi"    # Max 128 Mi before OOM
        cpu: "500m"        # Max 0.5 CPU (throttled if exceeded)
```

### QoS Classes

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       QUALITY OF SERVICE CLASSES                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Class       │ Condition                        │ Priority (eviction)              │
│  ────────────┼──────────────────────────────────┼──────────────────────────────────│
│  Guaranteed  │ requests == limits for all       │ Highest (last to be evicted)    │
│              │ containers and all resources     │                                  │
│  ────────────┼──────────────────────────────────┼──────────────────────────────────│
│  Burstable   │ At least one request or limit    │ Medium                          │
│              │ but not Guaranteed               │                                  │
│  ────────────┼──────────────────────────────────┼──────────────────────────────────│
│  BestEffort  │ No requests or limits set        │ Lowest (first to be evicted)    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏥 Health Probes

### Probe Types

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           HEALTH PROBES                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Probe Type      │ Purpose                           │ Action on Failure            │
│  ────────────────┼───────────────────────────────────┼──────────────────────────────│
│  livenessProbe   │ Is container alive?               │ Restart container            │
│                  │ Detects deadlock, hung process    │                              │
│  ────────────────┼───────────────────────────────────┼──────────────────────────────│
│  readinessProbe  │ Is container ready for traffic?   │ Remove from Service          │
│                  │ Detects startup, dependencies     │ (stop sending traffic)       │
│  ────────────────┼───────────────────────────────────┼──────────────────────────────│
│  startupProbe    │ Has container started?            │ Kill container               │
│                  │ For slow-starting containers      │ (liveness/readiness paused)  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Probe Methods

```yaml
# HTTP Probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome
  initialDelaySeconds: 10    # Wait before first probe
  periodSeconds: 5           # How often to probe
  timeoutSeconds: 1          # Timeout for probe
  successThreshold: 1        # Min consecutive successes
  failureThreshold: 3        # Failures before action

# TCP Probe
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 10

# Command Probe
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5

# gRPC Probe (K8s 1.24+)
readinessProbe:
  grpc:
    port: 50051
  initialDelaySeconds: 10
```

### Complete Probe Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-demo
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    
    # Startup probe - for slow starters
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30      # 30 * 10s = 5 minutes to start
      periodSeconds: 10
    
    # Liveness probe - is it alive?
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0    # Start after startup probe passes
      periodSeconds: 10
      failureThreshold: 3
    
    # Readiness probe - is it ready for traffic?
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 3
```

---

## 🔧 Environment Variables

### Setting Environment Variables

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    # Static value
    - name: NODE_ENV
      value: "production"
    
    # From ConfigMap key
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db.host
    
    # From Secret key
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secrets
          key: password
    
    # From Pod metadata
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    
    # From container resources
    - name: MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
    
    # All keys from ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
    # All keys from Secret
    - secretRef:
        name: app-secrets
```

---

## 💾 Volumes in Pods

### Common Volume Types

```yaml
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data
    - name: config
      mountPath: /etc/config
      readOnly: true
    - name: cache
      mountPath: /cache
    - name: secret
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  # EmptyDir - shared between containers, deleted when pod dies
  - name: cache
    emptyDir: {}
  
  # EmptyDir with memory
  - name: temp
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi
  
  # ConfigMap as volume
  - name: config
    configMap:
      name: app-config
      items:
      - key: config.yaml
        path: config.yaml
  
  # Secret as volume
  - name: secret
    secret:
      secretName: app-secrets
      defaultMode: 0400
  
  # PersistentVolumeClaim
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
  
  # HostPath (use carefully!)
  - name: host-data
    hostPath:
      path: /var/data
      type: DirectoryOrCreate
```

---

## 🔒 Pod Security

### Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security
  securityContext:
    runAsUser: 1000           # Run as non-root
    runAsGroup: 3000          # Primary group
    fsGroup: 2000             # Volume ownership group
    runAsNonRoot: true        # Ensure non-root
  
  containers:
  - name: app
    image: myapp
    # Container-level security
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
```

---

## 📍 Pod Scheduling

### Node Selection

```yaml
spec:
  # Specific node (rarely used)
  nodeName: worker-1
  
  # Label-based selection
  nodeSelector:
    disktype: ssd
    environment: production
```

### Node Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory
```

### Tolerations

```yaml
spec:
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "dedicated"
    operator: "Equal"
    value: "special-user"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

---

## 🔧 Troubleshooting Pods

### Common Issues

| Status | Cause | Debug |
|--------|-------|-------|
| `Pending` | No schedulable nodes | `kubectl describe pod` |
| `ImagePullBackOff` | Wrong image or no access | Check image name, registry |
| `CrashLoopBackOff` | App crashes repeatedly | `kubectl logs --previous` |
| `RunContainerError` | Config/volume issues | `kubectl describe pod` |
| `OOMKilled` | Memory limit exceeded | Increase limits |

### Debug Commands

```bash
# Check pod status and events
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name>

# Execute into pod
kubectl exec -it <pod-name> -- /bin/sh

# Check resource usage
kubectl top pod <pod-name>

# View events
kubectl get events --field-selector=involvedObject.name=<pod-name>
```

---

## 🎓 CKA Exam Tips

### Quick Pod Creation

```bash
# Generate pod YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Pod with specific port
kubectl run nginx --image=nginx --port=80 --dry-run=client -o yaml

# Pod with labels
kubectl run nginx --image=nginx --labels="app=nginx,tier=frontend"

# Pod with resource requests
kubectl run nginx --image=nginx --dry-run=client -o yaml | \
  sed '/resources:/a\        requests:\n          cpu: "100m"\n          memory: "128Mi"'
```

### Common Exam Tasks

```bash
# Create pod with specific requirements
kubectl run nginx --image=nginx --port=80 --labels="app=nginx"

# Add environment variable
kubectl set env pod/nginx ENV=production

# Execute command
kubectl exec nginx -- ls /etc/nginx

# Copy file
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
```

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Pod** | Smallest deployable unit, wraps containers |
| **Multi-container** | Sidecar, ambassador, adapter patterns |
| **Init containers** | Run before app containers |
| **Probes** | Liveness, readiness, startup |
| **Resources** | Requests (guaranteed), limits (max) |
| **QoS** | Guaranteed, Burstable, BestEffort |

### Pod Lifecycle

```
Pending → Scheduled → Init Containers → Running → Succeeded/Failed
```

---

## 🔜 What's Next

In **Chapter 07: ReplicaSets**, we'll cover:

- Maintaining multiple pod replicas
- Selector and pod template
- Scaling and updates
- When to use ReplicaSets vs Deployments

---

*Pods are the foundation - understand them deeply before moving on!*

