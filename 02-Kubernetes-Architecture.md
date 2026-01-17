# ☸️ Chapter 02: Kubernetes Architecture

> A comprehensive deep-dive into every component of a Kubernetes cluster and how they work together.

---

## 📚 Table of Contents

1. [Cluster Overview](#-cluster-overview)
2. [Control Plane Components](#-control-plane-components)
3. [Worker Node Components](#-worker-node-components)
4. [Add-on Components](#-add-on-components)
5. [Communication Flow](#-communication-flow)
6. [High Availability Architecture](#-high-availability-architecture)
7. [Data Flow Examples](#-data-flow-examples)
8. [Component Ports and Protocols](#-component-ports-and-protocols)
9. [CKA Exam Deep Dive](#-cka-exam-deep-dive)
10. [Summary](#-summary)

---

## 🏗️ Cluster Overview

### What is a Kubernetes Cluster?

> **Definition**: A Kubernetes cluster is a set of machines (nodes) that run containerized applications managed by Kubernetes. It consists of at least one **Control Plane** (master) and one or more **Worker Nodes**.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           KUBERNETES CLUSTER ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                              CONTROL PLANE                                     │  │
│  │                         (The "Brain" of the cluster)                          │  │
│  │                                                                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ API Server  │  │    etcd     │  │  Scheduler  │  │ Controller Manager  │  │  │
│  │  │             │  │             │  │             │  │                     │  │  │
│  │  │ REST API    │  │ Key-Value   │  │ Pod         │  │ Multiple            │  │  │
│  │  │ Gateway     │  │ Store       │  │ Placement   │  │ Controllers         │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  │                                                                               │  │
│  │  Optional: ┌─────────────────────────┐                                       │  │
│  │            │ Cloud Controller Manager │  (For cloud providers)               │  │
│  │            └─────────────────────────┘                                       │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                         │                                            │
│                                         │ kubectl / API calls                        │
│                                         ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                              WORKER NODES                                      │  │
│  │                      (Where your applications run)                            │  │
│  │                                                                               │  │
│  │  ┌─────────────────────────────┐    ┌─────────────────────────────┐          │  │
│  │  │         Node 1              │    │         Node 2              │          │  │
│  │  │  ┌─────────┐ ┌───────────┐  │    │  ┌─────────┐ ┌───────────┐  │          │  │
│  │  │  │ kubelet │ │kube-proxy │  │    │  │ kubelet │ │kube-proxy │  │          │  │
│  │  │  └─────────┘ └───────────┘  │    │  └─────────┘ └───────────┘  │          │  │
│  │  │  ┌─────────────────────────┐│    │  ┌─────────────────────────┐│          │  │
│  │  │  │   Container Runtime     ││    │  │   Container Runtime     ││          │  │
│  │  │  │     (containerd)        ││    │  │     (containerd)        ││          │  │
│  │  │  └─────────────────────────┘│    │  └─────────────────────────┘│          │  │
│  │  │  ┌─────┐ ┌─────┐ ┌─────┐   │    │  ┌─────┐ ┌─────┐ ┌─────┐   │          │  │
│  │  │  │ Pod │ │ Pod │ │ Pod │   │    │  │ Pod │ │ Pod │ │ Pod │   │          │  │
│  │  │  └─────┘ └─────┘ └─────┘   │    │  └─────┘ └─────┘ └─────┘   │          │  │
│  │  └─────────────────────────────┘    └─────────────────────────────┘          │  │
│  │                                                                               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Node Types

| Node Type | Also Called | Purpose | Quantity |
|-----------|-------------|---------|----------|
| **Control Plane** | Master Node | Manages cluster state, scheduling, API | 1 (dev), 3+ (prod HA) |
| **Worker Node** | Worker, Minion | Runs application workloads (Pods) | 1 to thousands |

### Real-World Analogy

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           AIRPORT ANALOGY                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Control Plane = Airport Control Tower                                               │
│  ├── API Server      = Main communication hub (radio tower)                        │
│  ├── etcd            = Flight database (all flight info stored)                    │
│  ├── Scheduler       = Gate assignment (decides where planes go)                   │
│  └── Controllers     = Airline managers (ensure right planes at gates)             │
│                                                                                      │
│  Worker Nodes = Airport Terminals                                                    │
│  ├── kubelet         = Terminal manager (manages gates/planes)                      │
│  ├── kube-proxy      = Signage system (directs passengers)                          │
│  ├── Container Runtime = Ground crew (actually moves planes)                        │
│  └── Pods            = Airplanes (carry the actual passengers/cargo)               │
│                                                                                      │
│  Passengers = Your application users                                                │
│  Cargo = Your application data                                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Control Plane Components

The Control Plane makes global decisions about the cluster (scheduling, detecting/responding to events).

### 1. kube-apiserver

> **Definition**: The API Server is the front-end of the Kubernetes control plane. It exposes the Kubernetes API and is the only component that communicates directly with etcd.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              kube-apiserver                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Central hub for ALL cluster communication                                 │
│                                                                                      │
│                    ┌─────────────────────────────────┐                              │
│                    │          API SERVER              │                              │
│                    │                                 │                              │
│                    │  • RESTful API                  │                              │
│                    │  • Authentication               │                              │
│                    │  • Authorization                │                              │
│                    │  • Admission Control            │                              │
│                    │  • Validation                   │                              │
│                    │  • etcd Communication           │                              │
│                    │                                 │                              │
│                    └───────────────┬─────────────────┘                              │
│                                    │                                                 │
│           ┌────────────────────────┼────────────────────────┐                       │
│           │                        │                        │                        │
│           ▼                        ▼                        ▼                        │
│    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐                │
│    │   kubectl   │          │   kubelet   │          │ Controllers │                │
│    │   (Users)   │          │  (Nodes)    │          │  (Internal) │                │
│    └─────────────┘          └─────────────┘          └─────────────┘                │
│                                                                                      │
│  Key Characteristics:                                                               │
│  • ONLY component that talks to etcd                                                │
│  • Stateless (can be horizontally scaled)                                           │
│  • Validates and configures API objects                                             │
│  • Serves REST operations                                                           │
│  • Default port: 6443 (HTTPS)                                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### API Server Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         API REQUEST PROCESSING PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Request: kubectl create deployment nginx --image=nginx                             │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │  1. AUTHENTICATION                                                          │   │
│  │     "Who are you?"                                                          │   │
│  │     • Client certificates                                                   │   │
│  │     • Bearer tokens                                                         │   │
│  │     • Service accounts                                                      │   │
│  │                              ▼                                              │   │
│  │  2. AUTHORIZATION                                                           │   │
│  │     "What can you do?"                                                      │   │
│  │     • RBAC (Role-Based Access Control)                                     │   │
│  │     • ABAC (Attribute-Based)                                               │   │
│  │     • Webhook                                                               │   │
│  │                              ▼                                              │   │
│  │  3. ADMISSION CONTROL                                                       │   │
│  │     "Should I allow this specific request?"                                │   │
│  │     • Mutating webhooks (modify request)                                   │   │
│  │     • Validating webhooks (accept/reject)                                  │   │
│  │     • Built-in admission controllers                                       │   │
│  │                              ▼                                              │   │
│  │  4. VALIDATION                                                              │   │
│  │     "Is this request valid?"                                               │   │
│  │     • Schema validation                                                     │   │
│  │     • Required fields                                                       │   │
│  │     • Type checking                                                         │   │
│  │                              ▼                                              │   │
│  │  5. PERSIST TO etcd                                                         │   │
│  │     "Store the object"                                                      │   │
│  │     • Write to etcd                                                         │   │
│  │     • Return response                                                       │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### Key API Server Flags (CKA Important)

```bash
# Common API server configuration (in /etc/kubernetes/manifests/kube-apiserver.yaml)
--advertise-address=192.168.1.10           # IP to advertise
--allow-privileged=true                     # Allow privileged containers
--authorization-mode=Node,RBAC             # Authorization modes
--client-ca-file=/etc/kubernetes/pki/ca.crt
--enable-admission-plugins=NodeRestriction
--etcd-servers=https://127.0.0.1:2379      # etcd connection
--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
--secure-port=6443                          # HTTPS port
--service-cluster-ip-range=10.96.0.0/12   # Service IP range
```

---

### 2. etcd

> **Definition**: etcd is a consistent, highly-available key-value store used as Kubernetes' backing store for all cluster data. It stores the entire cluster state.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    etcd                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: The "source of truth" for all cluster state                              │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │                          ┌───────────────────┐                               │ │
│  │                          │       etcd        │                               │ │
│  │                          │                   │                               │ │
│  │                          │  Key-Value Store  │                               │ │
│  │                          │                   │                               │ │
│  │                          │  /registry/       │                               │ │
│  │                          │  ├── pods/        │                               │ │
│  │                          │  ├── services/    │                               │ │
│  │                          │  ├── deployments/ │                               │ │
│  │                          │  ├── secrets/     │                               │ │
│  │                          │  ├── configmaps/  │                               │ │
│  │                          │  └── ...          │                               │ │
│  │                          └───────────────────┘                               │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  What's Stored in etcd:                                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ • All Kubernetes objects (Pods, Services, Deployments, Secrets, etc.)         ││
│  │ • Cluster configuration                                                        ││
│  │ • Current state of all resources                                              ││
│  │ • Service accounts and RBAC data                                              ││
│  │ • Lease information                                                            ││
│  │ • Custom Resource Definitions (CRDs)                                          ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  Key Characteristics:                                                               │
│  • Distributed (runs on multiple nodes for HA)                                     │
│  • Consistent (uses Raft consensus algorithm)                                      │
│  • Highly available (survives node failures)                                       │
│  • Default port: 2379 (client), 2380 (peer)                                        │
│  • ONLY API Server communicates with etcd                                          │
│                                                                                      │
│  ⚠️  CRITICAL: etcd is the ONLY stateful component!                               │
│      Lose etcd = Lose your cluster state                                           │
│      Always backup etcd!                                                            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### etcd Commands (CKA Essential)

```bash
# Check etcd cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# List all keys (see what's stored)
ETCDCTL_API=3 etcdctl get / --prefix --keys-only

# Backup etcd (CRITICAL for CKA!)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Restore etcd (CKA scenario!)
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

#### etcd Data Structure

```
/registry/
├── pods/
│   └── default/
│       ├── nginx-deployment-abc123/
│       └── my-app-def456/
├── services/
│   ├── default/
│   │   ├── kubernetes/
│   │   └── my-service/
│   └── kube-system/
│       └── kube-dns/
├── deployments/
│   └── default/
│       └── nginx-deployment/
├── secrets/
│   └── default/
│       └── my-secret/
├── configmaps/
│   └── default/
│       └── my-config/
└── ...
```

---

### 3. kube-scheduler

> **Definition**: The Scheduler watches for newly created Pods with no assigned node, and selects a node for them to run on based on resource requirements, constraints, and policies.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              kube-scheduler                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Decides WHICH node a Pod should run on                                    │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                         SCHEDULING PROCESS                                     │ │
│  │                                                                               │ │
│  │   1. Watch for unscheduled pods                                               │ │
│  │      (pods with spec.nodeName = empty)                                       │ │
│  │                                                                               │ │
│  │                         ▼                                                     │ │
│  │                                                                               │ │
│  │   2. FILTERING (find feasible nodes)                                         │ │
│  │      ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │      │ • Does node have enough resources? (CPU, Memory)                    │ │ │
│  │      │ • Does node match nodeSelector?                                     │ │ │
│  │      │ • Does node satisfy affinity rules?                                 │ │ │
│  │      │ • Does node have required taints/tolerations?                       │ │ │
│  │      │ • Is node ready and schedulable?                                    │ │ │
│  │      └─────────────────────────────────────────────────────────────────────┘ │ │
│  │                         ▼                                                     │ │
│  │                                                                               │ │
│  │   3. SCORING (rank remaining nodes)                                          │ │
│  │      ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │      │ • Spread pods across nodes/zones                                    │ │ │
│  │      │ • Prefer nodes with image already cached                           │ │ │
│  │      │ • Balance resource utilization                                      │ │ │
│  │      │ • User-defined priorities                                           │ │ │
│  │      └─────────────────────────────────────────────────────────────────────┘ │ │
│  │                         ▼                                                     │ │
│  │                                                                               │ │
│  │   4. BINDING (assign pod to best node)                                       │ │
│  │      Sets pod.spec.nodeName = selected-node                                  │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  Key Point: Scheduler ONLY decides where pods run.                                  │
│             It does NOT actually start the pod (kubelet does that).                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### Scheduling Decision Example

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SCHEDULING DECISION EXAMPLE                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Pod Requirements:                                                                   │
│  • CPU: 500m (0.5 cores)                                                            │
│  • Memory: 256Mi                                                                    │
│  • nodeSelector: disktype=ssd                                                       │
│                                                                                      │
│  Available Nodes:                                                                   │
│  ┌───────────────────┬───────────────────┬───────────────────┬─────────────────┐   │
│  │     Node 1        │     Node 2        │     Node 3        │     Result      │   │
│  ├───────────────────┼───────────────────┼───────────────────┼─────────────────┤   │
│  │ CPU: 2 cores      │ CPU: 4 cores      │ CPU: 1 core       │                 │   │
│  │ Free: 1.5 cores   │ Free: 0.3 cores   │ Free: 0.8 cores   │                 │   │
│  │ Memory: 4Gi       │ Memory: 8Gi       │ Memory: 2Gi       │                 │   │
│  │ Free: 2Gi         │ Free: 500Mi       │ Free: 1.5Gi       │                 │   │
│  │ Labels:           │ Labels:           │ Labels:           │                 │   │
│  │   disktype=ssd    │   disktype=hdd    │   disktype=ssd    │                 │   │
│  ├───────────────────┼───────────────────┼───────────────────┼─────────────────┤   │
│  │ Filtering:        │ Filtering:        │ Filtering:        │                 │   │
│  │ ✅ Has resources  │ ❌ Not enough CPU │ ✅ Has resources  │                 │   │
│  │ ✅ Has SSD label  │ ❌ Wrong disk type│ ✅ Has SSD label  │                 │   │
│  │                   │                   │                   │                 │   │
│  │ PASSED            │ FILTERED OUT      │ PASSED            │                 │   │
│  ├───────────────────┼───────────────────┼───────────────────┼─────────────────┤   │
│  │ Scoring:          │       N/A         │ Scoring:          │                 │   │
│  │ Score: 85         │                   │ Score: 72         │                 │   │
│  │ (more resources)  │                   │ (less resources)  │                 │   │
│  ├───────────────────┼───────────────────┼───────────────────┼─────────────────┤   │
│  │ ⭐ SELECTED       │       ❌          │       ❌          │ Node 1 wins!    │   │
│  └───────────────────┴───────────────────┴───────────────────┴─────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### Scheduler Plugins and Policies

```yaml
# Influencing Scheduling Decisions (in Pod spec)

# 1. nodeSelector (simple)
spec:
  nodeSelector:
    disktype: ssd
    environment: production

# 2. Node Affinity (advanced)
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b

# 3. Pod Anti-Affinity (spread pods)
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname

# 4. Tolerations (for taints)
spec:
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

---

### 4. kube-controller-manager

> **Definition**: The Controller Manager runs controller processes. Each controller is a separate process, but to reduce complexity, they are compiled into a single binary and run in a single process.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           kube-controller-manager                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Runs controllers that regulate the state of the cluster                   │
│                                                                                      │
│  What is a Controller?                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ A control loop that watches the current state of the cluster through the    │   │
│  │ API Server and makes changes to move the current state toward desired state │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                      CONTROLLER MANAGER                                        │ │
│  │                                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │                    BUILT-IN CONTROLLERS                                 │ │ │
│  │  ├─────────────────────────────────────────────────────────────────────────┤ │ │
│  │  │                                                                         │ │ │
│  │  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │ │ │
│  │  │  │ Node Controller   │  │Replication Ctrl   │  │ Endpoints Ctrl    │   │ │ │
│  │  │  │                   │  │                   │  │                   │   │ │ │
│  │  │  │ • Node health     │  │ • Maintains       │  │ • Populates       │   │ │ │
│  │  │  │ • Node lifecycle  │  │   correct # of    │  │   Endpoints       │   │ │ │
│  │  │  │ • CIDR assignment │  │   pod replicas    │  │   objects         │   │ │ │
│  │  │  └───────────────────┘  └───────────────────┘  └───────────────────┘   │ │ │
│  │  │                                                                         │ │ │
│  │  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │ │ │
│  │  │  │ SA Controller     │  │ Deployment Ctrl   │  │ StatefulSet Ctrl  │   │ │ │
│  │  │  │                   │  │                   │  │                   │   │ │ │
│  │  │  │ • Creates default │  │ • Manages         │  │ • Manages         │   │ │ │
│  │  │  │   service accounts│  │   ReplicaSets     │  │   stateful pods   │   │ │ │
│  │  │  │                   │  │ • Rolling updates │  │   with identity   │   │ │ │
│  │  │  └───────────────────┘  └───────────────────┘  └───────────────────┘   │ │ │
│  │  │                                                                         │ │ │
│  │  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │ │ │
│  │  │  │ Job Controller    │  │ CronJob Ctrl      │  │ DaemonSet Ctrl    │   │ │ │
│  │  │  │                   │  │                   │  │                   │   │ │ │
│  │  │  │ • Runs pods to    │  │ • Schedules jobs  │  │ • Ensures pod on  │   │ │ │
│  │  │  │   completion      │  │   based on cron   │  │   every node      │   │ │ │
│  │  │  └───────────────────┘  └───────────────────┘  └───────────────────┘   │ │ │
│  │  │                                                                         │ │ │
│  │  │  And many more: Namespace, ResourceQuota, PV, PVC, TTL, etc.           │ │ │
│  │  │                                                                         │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### Controller Loop (Reconciliation)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         CONTROLLER RECONCILIATION LOOP                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Example: ReplicaSet Controller                                                      │
│  Desired: 3 replicas                                                                │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │                    ┌─────────────────────────┐                              │   │
│  │              ┌────>│     Watch API Server    │<────┐                        │   │
│  │              │     │  for ReplicaSet changes │     │                        │   │
│  │              │     └───────────┬─────────────┘     │                        │   │
│  │              │                 │                    │                        │   │
│  │              │                 ▼                    │                        │   │
│  │              │     ┌─────────────────────────┐     │                        │   │
│  │              │     │   Get Current State     │     │                        │   │
│  │              │     │   (How many pods now?)  │     │                        │   │
│  │              │     └───────────┬─────────────┘     │                        │   │
│  │              │                 │                    │                        │   │
│  │              │                 ▼                    │                        │   │
│  │              │     ┌─────────────────────────┐     │                        │   │
│  │              │     │  Compare to Desired     │     │                        │   │
│  │              │     │  (Should be 3 pods)     │     │                        │   │
│  │              │     └───────────┬─────────────┘     │                        │   │
│  │              │                 │                    │                        │   │
│  │              │     ┌───────────┴───────────┐       │                        │   │
│  │              │     │                       │       │                        │   │
│  │              │     ▼                       ▼       │                        │   │
│  │              │  Current < 3            Current > 3  │                        │   │
│  │              │  ┌──────────┐           ┌──────────┐│                        │   │
│  │              │  │ CREATE   │           │ DELETE   ││                        │   │
│  │              │  │ more pods│           │ excess   ││                        │   │
│  │              │  └──────────┘           │ pods     ││                        │   │
│  │              │         │               └──────────┘│                        │   │
│  │              │         │                    │      │                        │   │
│  │              │         └────────┬───────────┘      │                        │   │
│  │              │                  │                   │                        │   │
│  │              │                  ▼                   │                        │   │
│  │              │     ┌─────────────────────────┐     │                        │   │
│  │              │     │    Update API Server    │     │                        │   │
│  │              │     └───────────┬─────────────┘     │                        │   │
│  │              │                 │                   │                        │   │
│  │              └─────────────────┴───────────────────┘                        │   │
│  │                           (Loop continues)                                  │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 5. cloud-controller-manager (Optional)

> **Definition**: The Cloud Controller Manager embeds cloud-specific control logic. It lets you link your cluster to your cloud provider's API, separating out components that interact with the cloud from components that only interact with your cluster.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          cloud-controller-manager                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Integrate with cloud provider APIs                                        │
│                                                                                      │
│  Only exists in cloud deployments (EKS, GKE, AKS, etc.)                            │
│                                                                                      │
│  Controllers:                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐       │   │
│  │  │ Node Controller   │  │ Route Controller  │  │ Service Ctrl      │       │   │
│  │  │                   │  │                   │  │                   │       │   │
│  │  │ • Checks if node  │  │ • Sets up routes  │  │ • Creates cloud   │       │   │
│  │  │   still exists in │  │   in cloud infra  │  │   load balancers  │       │   │
│  │  │   cloud provider  │  │                   │  │   for Service     │       │   │
│  │  │ • Deletes node    │  │                   │  │   type=LB         │       │   │
│  │  │   object if VM    │  │                   │  │                   │       │   │
│  │  │   is terminated   │  │                   │  │                   │       │   │
│  │  └───────────────────┘  └───────────────────┘  └───────────────────┘       │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Examples:                                                                          │
│  • AWS: Creates ELB when Service type=LoadBalancer                                 │
│  • GCP: Creates GCP Load Balancer                                                  │
│  • Azure: Creates Azure Load Balancer                                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 💪 Worker Node Components

Worker nodes run your application workloads (Pods).

### 1. kubelet

> **Definition**: The kubelet is an agent that runs on each node in the cluster. It ensures that containers are running in a Pod as specified by PodSpecs.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   kubelet                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: The "node agent" - ensures pods are running correctly on the node        │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │                           ┌───────────────────┐                               │ │
│  │                           │     API SERVER    │                               │ │
│  │                           └─────────┬─────────┘                               │ │
│  │                                     │                                          │ │
│  │           ┌─────────────────────────┼─────────────────────────┐               │ │
│  │           │                         ▼                         │               │ │
│  │           │              ┌───────────────────┐                │               │ │
│  │           │              │      kubelet      │                │               │ │
│  │           │              │                   │                │               │ │
│  │           │              │  • Watches for    │                │               │ │
│  │           │              │    pod specs      │                │               │ │
│  │           │              │  • Reports node   │                │               │ │
│  │           │              │    status         │                │               │ │
│  │           │              │  • Runs probes    │                │               │ │
│  │           │              │  • Mounts volumes │                │               │ │
│  │           │              └─────────┬─────────┘                │               │ │
│  │           │                        │                          │               │ │
│  │           │                        │ CRI (Container Runtime   │               │ │
│  │           │                        │      Interface)          │               │ │
│  │           │                        ▼                          │               │ │
│  │           │              ┌───────────────────┐                │               │ │
│  │           │              │ Container Runtime │                │               │ │
│  │           │              │   (containerd)    │                │               │ │
│  │           │              └─────────┬─────────┘                │               │ │
│  │           │                        │                          │               │ │
│  │           │                        ▼                          │               │ │
│  │           │        ┌─────┐    ┌─────┐    ┌─────┐             │               │ │
│  │           │        │ Pod │    │ Pod │    │ Pod │             │  NODE         │ │
│  │           │        └─────┘    └─────┘    └─────┘             │               │ │
│  │           │                                                   │               │ │
│  │           └───────────────────────────────────────────────────┘               │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  kubelet Responsibilities:                                                          │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ 1. Register node with API Server                                              ││
│  │ 2. Watch for Pods scheduled to this node                                      ││
│  │ 3. Create/start/stop containers via container runtime                         ││
│  │ 4. Mount volumes (secrets, configmaps, PVs)                                   ││
│  │ 5. Report node and pod status to API Server                                   ││
│  │ 6. Run liveness/readiness/startup probes                                      ││
│  │ 7. Garbage collection of images and containers                                ││
│  │ 8. Static pod management (for control plane pods)                             ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  Key Characteristics:                                                               │
│  • Runs as a system service (systemd) on each node                                 │
│  • NOT a Pod - runs directly on the node OS                                        │
│  • Default port: 10250 (HTTPS API)                                                 │
│  • Config typically at /var/lib/kubelet/config.yaml                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### kubelet Configuration (CKA Important)

```bash
# View kubelet configuration
cat /var/lib/kubelet/config.yaml

# Common kubelet flags
--config=/var/lib/kubelet/config.yaml
--container-runtime-endpoint=unix:///run/containerd/containerd.sock
--kubeconfig=/etc/kubernetes/kubelet.conf
--node-ip=192.168.1.20

# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f

# Restart kubelet
sudo systemctl restart kubelet
```

---

### 2. kube-proxy

> **Definition**: kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept. It maintains network rules that allow network communication to your Pods.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  kube-proxy                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Enable Service abstraction by routing traffic to backend Pods            │
│                                                                                      │
│  How Services Work (kube-proxy makes this possible):                               │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │   Client Pod                                                                  │ │
│  │       │                                                                       │ │
│  │       │ Connects to: my-service:80                                           │ │
│  │       │              (ClusterIP: 10.96.1.50)                                 │ │
│  │       ▼                                                                       │ │
│  │   ┌───────────────────────────────────────────────────────────────────┐      │ │
│  │   │                        kube-proxy                                 │      │ │
│  │   │                                                                   │      │ │
│  │   │  Watches API Server for Service/Endpoints changes                │      │ │
│  │   │  Creates iptables/IPVS rules to route traffic                    │      │ │
│  │   │                                                                   │      │ │
│  │   │  Rule: 10.96.1.50:80 → [Pod1:8080, Pod2:8080, Pod3:8080]        │      │ │
│  │   │                                                                   │      │ │
│  │   └───────────────────────────────────────────────────────────────────┘      │ │
│  │                │                                                              │ │
│  │   ┌────────────┼────────────┬────────────┐                                   │ │
│  │   │            │            │            │                                    │ │
│  │   ▼            ▼            ▼            │                                    │ │
│  │ ┌─────┐     ┌─────┐     ┌─────┐         │   (Load balanced across pods)     │ │
│  │ │Pod 1│     │Pod 2│     │Pod 3│         │                                    │ │
│  │ │:8080│     │:8080│     │:8080│         │                                    │ │
│  │ └─────┘     └─────┘     └─────┘         │                                    │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  Proxy Modes:                                                                       │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ Mode     │ Description                              │ Performance            ││
│  │──────────┼──────────────────────────────────────────┼────────────────────────││
│  │ iptables │ Default. Uses iptables rules for routing │ Good, O(n) rules      ││
│  │ IPVS     │ Uses IPVS (IP Virtual Server) module     │ Better, O(1) lookup   ││
│  │ userspace│ Legacy, user-space proxy                 │ Poor, deprecated      ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  Key Characteristics:                                                               │
│  • Runs as a DaemonSet (one pod per node)                                          │
│  • Watches Service and Endpoints objects                                           │
│  • Creates/updates network rules in real-time                                      │
│  • Enables ClusterIP, NodePort, and LoadBalancer services                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### kube-proxy iptables Example

```bash
# View iptables rules created by kube-proxy
sudo iptables -t nat -L KUBE-SERVICES -n

# Example output (simplified):
# Chain KUBE-SERVICES
# target     prot opt source   destination
# KUBE-SVC-XXXX  tcp  --  0.0.0.0/0  10.96.1.50   /* default/my-service:http */

# The KUBE-SVC chain load balances to endpoints:
sudo iptables -t nat -L KUBE-SVC-XXXX -n
# KUBE-SEP-AAAA  all  --  0.0.0.0/0  0.0.0.0/0  statistic mode random probability 0.33
# KUBE-SEP-BBBB  all  --  0.0.0.0/0  0.0.0.0/0  statistic mode random probability 0.50
# KUBE-SEP-CCCC  all  --  0.0.0.0/0  0.0.0.0/0
```

---

### 3. Container Runtime

> **Definition**: The container runtime is the software responsible for running containers. Kubernetes supports several runtimes through the Container Runtime Interface (CRI).

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              Container Runtime                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Actually run containers (Kubernetes just orchestrates)                    │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │                          ┌───────────────────┐                               │ │
│  │                          │      kubelet      │                               │ │
│  │                          └─────────┬─────────┘                               │ │
│  │                                    │                                          │ │
│  │                                    │ CRI (Container Runtime Interface)       │ │
│  │                                    │ gRPC API                                 │ │
│  │                                    │                                          │ │
│  │             ┌──────────────────────┼──────────────────────┐                  │ │
│  │             │                      │                      │                   │ │
│  │             ▼                      ▼                      ▼                   │ │
│  │    ┌───────────────┐     ┌───────────────┐     ┌───────────────┐            │ │
│  │    │  containerd   │     │    CRI-O      │     │    Docker     │            │ │
│  │    │  (Default)    │     │   (Red Hat)   │     │  (Deprecated) │            │ │
│  │    └───────┬───────┘     └───────┬───────┘     └───────┬───────┘            │ │
│  │            │                     │                     │                     │ │
│  │            │                     │                     │                     │ │
│  │            ▼                     ▼                     ▼                     │ │
│  │    ┌───────────────────────────────────────────────────────────────────┐    │ │
│  │    │                         runc                                      │    │ │
│  │    │              (Low-level OCI runtime)                             │    │ │
│  │    │                                                                   │    │ │
│  │    │   Actually creates containers using Linux kernel features:       │    │ │
│  │    │   • Namespaces (isolation)                                       │    │ │
│  │    │   • Cgroups (resource limits)                                    │    │ │
│  │    │   • Seccomp (syscall filtering)                                  │    │ │
│  │    └───────────────────────────────────────────────────────────────────┘    │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  Supported Runtimes:                                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ Runtime    │ Description                              │ Status               ││
│  │────────────┼──────────────────────────────────────────┼──────────────────────││
│  │ containerd │ Industry standard, CNCF graduated        │ Default, recommended ││
│  │ CRI-O      │ Lightweight for Kubernetes only          │ Supported            ││
│  │ Docker     │ Via dockershim (removed in K8s 1.24)    │ Deprecated           ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  Note: Docker images still work! Only Docker as a runtime is deprecated.           │
│        containerd can still pull and run Docker images.                            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### Check Container Runtime

```bash
# Check what runtime is being used
kubectl get nodes -o wide
# Look at CONTAINER-RUNTIME column

# Check containerd
crictl info

# List containers via crictl
crictl ps

# View containerd socket
ls -la /run/containerd/containerd.sock
```

---

## 🔌 Add-on Components

Add-ons extend Kubernetes functionality. They run as Pods in the cluster.

### CoreDNS (DNS)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  CoreDNS                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Purpose: Provides DNS-based service discovery within the cluster                   │
│                                                                                      │
│  How it works:                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │  Pod wants to reach "my-service"                                           │   │
│  │                                                                             │   │
│  │  1. Pod makes DNS query: my-service.default.svc.cluster.local              │   │
│  │                             │                                               │   │
│  │                             ▼                                               │   │
│  │  2. Query goes to CoreDNS (configured in /etc/resolv.conf of pod)          │   │
│  │                             │                                               │   │
│  │                             ▼                                               │   │
│  │  3. CoreDNS looks up the Service in its cache                              │   │
│  │     (watches API Server for Service/Endpoints changes)                     │   │
│  │                             │                                               │   │
│  │                             ▼                                               │   │
│  │  4. Returns ClusterIP: 10.96.1.50                                          │   │
│  │                             │                                               │   │
│  │                             ▼                                               │   │
│  │  5. Pod connects to 10.96.1.50:80                                          │   │
│  │     (kube-proxy routes to backend pods)                                    │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  DNS Record Formats:                                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ Type     │ Format                              │ Example                       ││
│  │──────────┼─────────────────────────────────────┼───────────────────────────────││
│  │ Service  │ <svc>.<ns>.svc.cluster.local        │ my-svc.default.svc.cluster... ││
│  │ Pod      │ <pod-ip>.<ns>.pod.cluster.local     │ 10-1-2-3.default.pod.clust... ││
│  │ Headless │ <pod>.<svc>.<ns>.svc.cluster.local  │ pod-0.my-svc.default.svc...   ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  Location: kube-system namespace                                                    │
│  Deploy type: Deployment with 2 replicas (HA)                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS resolution from a pod
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes
```

### Other Common Add-ons

| Add-on | Purpose |
|--------|---------|
| **Metrics Server** | Resource metrics (CPU, memory) for HPA |
| **Dashboard** | Web UI for cluster management |
| **Ingress Controller** | HTTP/HTTPS routing (nginx, traefik) |
| **CNI Plugin** | Pod networking (Calico, Flannel, Cilium) |
| **Cert-Manager** | TLS certificate management |
| **External-DNS** | External DNS record management |

---

## 🔄 Communication Flow

### Complete Request Flow: Creating a Deployment

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│           COMPLETE FLOW: kubectl create deployment nginx --image=nginx              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   STEP 1: kubectl sends request                                                      │
│   ┌─────────────┐                                                                   │
│   │  Developer  │                                                                   │
│   │  kubectl    │─────── HTTPS ──────────────────────────────────────┐              │
│   └─────────────┘                                                     │              │
│                                                                       ▼              │
│   STEP 2: API Server authenticates, authorizes, validates                          │
│                                            ┌───────────────────────────────────────┐│
│                                            │           API SERVER                  ││
│                                            │  • Authenticate (who are you?)       ││
│                                            │  • Authorize (can you do this?)      ││
│                                            │  • Admission control (any policies?) ││
│                                            │  • Validate (is spec correct?)       ││
│                                            └───────────────────┬───────────────────┘│
│                                                                │                     │
│   STEP 3: API Server writes to etcd                           ▼                     │
│                                            ┌───────────────────────────────────────┐│
│                                            │              etcd                     ││
│                                            │  Stores: Deployment object            ││
│                                            │          (desired state)              ││
│                                            └───────────────────┬───────────────────┘│
│                                                                │                     │
│   STEP 4: Deployment Controller notices new Deployment        │                     │
│                                            ┌───────────────────▼───────────────────┐│
│                                            │     DEPLOYMENT CONTROLLER             ││
│                                            │  "New Deployment! I'll create a      ││
│                                            │   ReplicaSet for it"                  ││
│                                            └───────────────────┬───────────────────┘│
│                                                                │                     │
│   STEP 5: ReplicaSet Controller creates Pods                  ▼                     │
│                                            ┌───────────────────────────────────────┐│
│                                            │     REPLICASET CONTROLLER             ││
│                                            │  "ReplicaSet says 1 replica needed.  ││
│                                            │   Creating Pod..."                    ││
│                                            └───────────────────┬───────────────────┘│
│                                                                │                     │
│   STEP 6: Scheduler assigns Pod to a Node                     ▼                     │
│                                            ┌───────────────────────────────────────┐│
│                                            │          SCHEDULER                    ││
│                                            │  "New Pod with no node! Let me find  ││
│                                            │   the best node... Node-1 looks good"││
│                                            │  Sets: pod.spec.nodeName = node-1    ││
│                                            └───────────────────┬───────────────────┘│
│                                                                │                     │
│   STEP 7: kubelet on Node-1 notices Pod assigned to it        ▼                     │
│   ┌───────────────────────────────────────────────────────────────────────────────┐│
│   │                              NODE-1                                            ││
│   │  ┌─────────────────────────────────────────────────────────────────────────┐  ││
│   │  │                            kubelet                                      │  ││
│   │  │  "A Pod is assigned to me! Let me start it..."                         │  ││
│   │  │  • Pulls image (nginx) via containerd                                  │  ││
│   │  │  • Creates container                                                    │  ││
│   │  │  • Mounts volumes                                                       │  ││
│   │  │  • Sets up networking                                                   │  ││
│   │  │  • Reports status back to API Server                                   │  ││
│   │  └─────────────────────────────────────────────────────────────────────────┘  ││
│   │                                                                                ││
│   │  ┌─────────────────────────────────────────────────────────────────────────┐  ││
│   │  │                         CONTAINER                                       │  ││
│   │  │                          nginx                                          │  ││
│   │  │                         RUNNING ✅                                      │  ││
│   │  └─────────────────────────────────────────────────────────────────────────┘  ││
│   └───────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│   RESULT: Deployment → ReplicaSet → Pod → Container                                │
│           All managed by different controllers working together                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 High Availability Architecture

For production, you need multiple control plane nodes:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     HIGH AVAILABILITY KUBERNETES CLUSTER                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│                              ┌───────────────────┐                                  │
│                              │   Load Balancer   │                                  │
│                              │  (HAProxy/NLB)    │                                  │
│                              └─────────┬─────────┘                                  │
│                                        │                                             │
│              ┌─────────────────────────┼─────────────────────────┐                  │
│              │                         │                         │                   │
│              ▼                         ▼                         ▼                   │
│  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐             │
│  │ Control Plane 1   │   │ Control Plane 2   │   │ Control Plane 3   │             │
│  │                   │   │                   │   │                   │             │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │             │
│  │ │ API Server    │ │   │ │ API Server    │ │   │ │ API Server    │ │             │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │             │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │             │
│  │ │ Controller    │ │   │ │ Controller    │ │   │ │ Controller    │ │             │
│  │ │ Manager       │ │   │ │ Manager       │ │   │ │ Manager       │ │             │
│  │ │ (Active)      │ │   │ │ (Standby)     │ │   │ │ (Standby)     │ │             │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │             │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │             │
│  │ │ Scheduler     │ │   │ │ Scheduler     │ │   │ │ Scheduler     │ │             │
│  │ │ (Active)      │ │   │ │ (Standby)     │ │   │ │ (Standby)     │ │             │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │             │
│  │ ┌───────────────┐ │   │ ┌───────────────┐ │   │ ┌───────────────┐ │             │
│  │ │ etcd          │◀───▶│ etcd          │◀───▶│ etcd          │ │             │
│  │ │ (Raft Leader) │ │   │ │ (Follower)    │ │   │ │ (Follower)    │ │             │
│  │ └───────────────┘ │   │ └───────────────┘ │   │ └───────────────┘ │             │
│  └───────────────────┘   └───────────────────┘   └───────────────────┘             │
│                                                                                      │
│  HA Characteristics:                                                                │
│  • API Servers: All active (stateless, load balanced)                              │
│  • Controllers/Scheduler: Leader election (only one active)                        │
│  • etcd: Raft consensus (needs quorum: 2 of 3, 3 of 5, etc.)                      │
│                                                                                      │
│  Odd numbers for etcd!                                                              │
│  • 3 nodes: Tolerates 1 failure (quorum = 2)                                       │
│  • 5 nodes: Tolerates 2 failures (quorum = 3)                                      │
│  • 7 nodes: Tolerates 3 failures (quorum = 4)                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔌 Component Ports and Protocols

### CKA Essential: Know These Ports!

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           KUBERNETES PORTS                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  CONTROL PLANE PORTS:                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ Component         │ Port      │ Protocol │ Purpose                             ││
│  │───────────────────┼───────────┼──────────┼─────────────────────────────────────││
│  │ API Server        │ 6443      │ HTTPS    │ Kubernetes API                      ││
│  │ etcd              │ 2379      │ HTTPS    │ Client connections                  ││
│  │ etcd              │ 2380      │ HTTPS    │ Peer communication                  ││
│  │ Scheduler         │ 10259     │ HTTPS    │ Metrics/health                      ││
│  │ Controller Manager│ 10257     │ HTTPS    │ Metrics/health                      ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  WORKER NODE PORTS:                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ Component         │ Port      │ Protocol │ Purpose                             ││
│  │───────────────────┼───────────┼──────────┼─────────────────────────────────────││
│  │ kubelet           │ 10250     │ HTTPS    │ kubelet API                         ││
│  │ kubelet (read)    │ 10255     │ HTTP     │ Read-only (deprecated)              ││
│  │ kube-proxy        │ 10256     │ HTTP     │ Health check                        ││
│  │ NodePort Services │ 30000-32767│ TCP/UDP │ External access                     ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  ADD-ONS:                                                                           │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │ Component         │ Port      │ Protocol │ Purpose                             ││
│  │───────────────────┼───────────┼──────────┼─────────────────────────────────────││
│  │ CoreDNS           │ 53        │ TCP/UDP  │ DNS resolution                      ││
│  │ Metrics Server    │ 443       │ HTTPS    │ Metrics API                         ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🎓 CKA Exam Deep Dive

### Common CKA Questions on Architecture

| Question Type | Example | Key Knowledge |
|---------------|---------|---------------|
| **etcd backup/restore** | "Backup etcd to /backup/etcd.db" | ETCDCTL_API=3, certs location |
| **kubelet troubleshooting** | "Node not ready, fix it" | systemctl, journalctl, config |
| **Static pods** | "Create a static pod" | /etc/kubernetes/manifests |
| **Cluster component check** | "Which component schedules pods?" | Scheduler |
| **Control plane certs** | "Check when certs expire" | kubeadm certs check-expiration |

### CKA Cheat Sheet: Architecture

```bash
# View cluster components
kubectl get componentstatuses  # Deprecated but may appear
kubectl get nodes -o wide

# Check control plane pods
kubectl get pods -n kube-system

# View API Server settings
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check scheduler config
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Check controller manager
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Check etcd
cat /etc/kubernetes/manifests/etcd.yaml

# Static pod location
ls /etc/kubernetes/manifests/

# kubelet config
cat /var/lib/kubelet/config.yaml

# kubelet status
systemctl status kubelet
journalctl -u kubelet -f

# kube-proxy mode
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep "Using"
```

---

## ✅ Summary

### Component Quick Reference

| Component | Location | Purpose | Port |
|-----------|----------|---------|------|
| **API Server** | Control Plane | REST API, authentication | 6443 |
| **etcd** | Control Plane | Cluster state storage | 2379 |
| **Scheduler** | Control Plane | Pod placement | 10259 |
| **Controller Manager** | Control Plane | Reconciliation loops | 10257 |
| **kubelet** | Every Node | Pod lifecycle | 10250 |
| **kube-proxy** | Every Node | Service networking | 10256 |
| **Container Runtime** | Every Node | Run containers | N/A |
| **CoreDNS** | Add-on (Pod) | Service discovery | 53 |

### Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE KEY POINTS                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. API Server is the ONLY component that talks to etcd                             │
│                                                                                      │
│  2. etcd is the ONLY stateful component - ALWAYS backup it!                         │
│                                                                                      │
│  3. Controllers run reconciliation loops (desired vs current state)                 │
│                                                                                      │
│  4. Scheduler ONLY decides where pods run, kubelet actually starts them            │
│                                                                                      │
│  5. kubelet runs as a systemd service, NOT as a pod                                 │
│                                                                                      │
│  6. kube-proxy enables the Service abstraction via iptables/IPVS                   │
│                                                                                      │
│  7. For HA: API Servers are load balanced, others use leader election              │
│                                                                                      │
│  8. etcd needs odd number of nodes for proper quorum                               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔜 What's Next

In **Chapter 03: Installing Kubernetes**, we'll:

- Set up a local Kubernetes cluster (Minikube, kind)
- Understand kubeadm installation
- Configure kubectl
- Explore cloud-managed options (EKS, GKE, AKS)

---

*Understanding the architecture is fundamental for troubleshooting - let's continue!*

