# Chapter 22: Scheduling - Pod Placement on Nodes

## Introduction

When you create a pod, Kubernetes decides which node to run it on. This decision is made by the **kube-scheduler**, and you can influence it through various mechanisms. Understanding scheduling is crucial for optimizing performance, ensuring high availability, and meeting compliance requirements.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE SCHEDULING PROBLEM                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   You have pods and nodes. Where should each pod run?                   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Pods Waiting:                    Available Nodes:             │   │
│   │   ┌─────┐ ┌─────┐ ┌─────┐         ┌──────────────┐             │   │
│   │   │ DB  │ │ Web │ │ API │         │   Node 1     │             │   │
│   │   │     │ │     │ │     │         │   (SSD, GPU) │             │   │
│   │   └─────┘ └─────┘ └─────┘         │   Zone: A    │             │   │
│   │                                    └──────────────┘             │   │
│   │                                    ┌──────────────┐             │   │
│   │   Requirements:                    │   Node 2     │             │   │
│   │   • DB needs SSD                   │   (HDD)      │             │   │
│   │   • Web spread across zones        │   Zone: B    │             │   │
│   │   • API near DB                    └──────────────┘             │   │
│   │                                    ┌──────────────┐             │   │
│   │                                    │   Node 3     │             │   │
│   │                                    │   (SSD)      │             │   │
│   │                                    │   Zone: A    │             │   │
│   │                                    └──────────────┘             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   How does Kubernetes decide?                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. How the Scheduler Works

### 1.1 Scheduling Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEDULER WORKFLOW                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      kube-scheduler                             │   │
│   │                                                                 │   │
│   │   1. WATCH for unscheduled pods                                 │   │
│   │      └── Pods with nodeName = empty                            │   │
│   │                │                                                │   │
│   │                ▼                                                │   │
│   │   2. FILTERING (Predicates)                                     │   │
│   │      Find nodes that CAN run the pod:                          │   │
│   │      ├── Enough CPU/memory?                                    │   │
│   │      ├── Node selectors match?                                 │   │
│   │      ├── Taints tolerated?                                     │   │
│   │      ├── Affinity rules satisfied?                             │   │
│   │      └── PV requirements met?                                  │   │
│   │                │                                                │   │
│   │                ▼                                                │   │
│   │   3. SCORING (Priorities)                                       │   │
│   │      Rank nodes by preference:                                 │   │
│   │      ├── Spread pods evenly                                    │   │
│   │      ├── Prefer nodes with images                              │   │
│   │      ├── Affinity preferences                                  │   │
│   │      └── Resource balance                                      │   │
│   │                │                                                │   │
│   │                ▼                                                │   │
│   │   4. BIND pod to highest-scoring node                          │   │
│   │      └── Sets pod.spec.nodeName                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   If no nodes pass filtering → Pod stays Pending                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Scheduling Mechanisms Overview

| Mechanism | Purpose | Type |
|-----------|---------|------|
| **nodeSelector** | Simple node selection by label | Hard requirement |
| **Node Affinity** | Advanced node selection rules | Hard or soft |
| **Pod Affinity** | Place pods together | Hard or soft |
| **Pod Anti-Affinity** | Keep pods apart | Hard or soft |
| **Taints/Tolerations** | Node repels pods unless tolerated | Hard restriction |
| **Topology Spread** | Distribute pods evenly | Hard or soft |
| **nodeName** | Direct assignment to node | Bypass scheduler |

---

## 2. nodeSelector

### 2.1 What is nodeSelector?

The simplest way to constrain pods to nodes with specific labels.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  nodeSelector:
    disktype: ssd              # Node must have this label
    environment: production
  containers:
  - name: db
    image: postgres
```

### 2.2 Node Labels

```bash
# View node labels
kubectl get nodes --show-labels

# Add label to node
kubectl label nodes node1 disktype=ssd

# Remove label from node
kubectl label nodes node1 disktype-

# Update existing label
kubectl label nodes node1 disktype=hdd --overwrite
```

### 2.3 Built-in Node Labels

| Label | Description |
|-------|-------------|
| `kubernetes.io/hostname` | Node hostname |
| `kubernetes.io/os` | Operating system (linux, windows) |
| `kubernetes.io/arch` | Architecture (amd64, arm64) |
| `topology.kubernetes.io/zone` | Cloud provider zone |
| `topology.kubernetes.io/region` | Cloud provider region |
| `node.kubernetes.io/instance-type` | Instance type (e.g., m5.large) |

```yaml
# Use built-in labels
spec:
  nodeSelector:
    kubernetes.io/os: linux
    topology.kubernetes.io/zone: us-east-1a
```

---

## 3. Node Affinity

### 3.1 What is Node Affinity?

**Node Affinity** is an advanced version of nodeSelector with more expressive rules.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NODE AFFINITY TYPES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   requiredDuringSchedulingIgnoredDuringExecution                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   "MUST" - Hard requirement                                     │   │
│   │                                                                 │   │
│   │   • Pod will NOT be scheduled unless condition is met          │   │
│   │   • Pod stays Pending if no matching node exists               │   │
│   │   • "IgnoredDuringExecution" = running pods stay if node       │   │
│   │     labels change                                              │   │
│   │                                                                 │   │
│   │   Use for: Critical constraints (zone, hardware type)          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   preferredDuringSchedulingIgnoredDuringExecution                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   "PREFER" - Soft preference                                    │   │
│   │                                                                 │   │
│   │   • Scheduler tries to match but will schedule anyway          │   │
│   │   • Uses weight (1-100) to rank preferences                    │   │
│   │   • Higher weight = stronger preference                        │   │
│   │                                                                 │   │
│   │   Use for: Optimization, not requirements                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Node Affinity Operators

| Operator | Meaning |
|----------|---------|
| `In` | Label value is in list |
| `NotIn` | Label value is not in list |
| `Exists` | Label exists (any value) |
| `DoesNotExist` | Label does not exist |
| `Gt` | Label value greater than (numeric) |
| `Lt` | Label value less than (numeric) |

### 3.3 Required Node Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: required-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
          - key: environment
            operator: NotIn
            values:
            - development
```

### 3.4 Preferred Node Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100              # Highest priority
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 50               # Lower priority
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: app
    image: nginx
```

### 3.5 Combined Required + Preferred

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: combined-affinity
spec:
  affinity:
    nodeAffinity:
      # MUST be on Linux
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      # PREFER SSD nodes
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx
```

---

## 4. Pod Affinity and Anti-Affinity

### 4.1 What is Pod Affinity?

**Pod Affinity** schedules pods based on labels of other pods already running on nodes.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POD AFFINITY vs ANTI-AFFINITY                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   POD AFFINITY: "Run near these pods"                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Use case: Frontend pods near backend pods (low latency)       │   │
│   │                                                                 │   │
│   │   ┌──────────────┐                                              │   │
│   │   │    Node 1    │                                              │   │
│   │   │  ┌────────┐  │                                              │   │
│   │   │  │Backend │  │  ◀── Frontend wants to be HERE              │   │
│   │   │  │ (app=  │  │                                              │   │
│   │   │  │backend)│  │                                              │   │
│   │   │  └────────┘  │                                              │   │
│   │   │  ┌────────┐  │                                              │   │
│   │   │  │Frontend│  │  ◀── Scheduled on same node                 │   │
│   │   │  └────────┘  │                                              │   │
│   │   └──────────────┘                                              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   POD ANTI-AFFINITY: "Run away from these pods"                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Use case: Spread replicas for high availability               │   │
│   │                                                                 │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │   │
│   │   │    Node 1    │  │    Node 2    │  │    Node 3    │         │   │
│   │   │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │         │   │
│   │   │  │  Web   │  │  │  │  Web   │  │  │  │  Web   │  │         │   │
│   │   │  │Replica1│  │  │  │Replica2│  │  │  │Replica3│  │         │   │
│   │   │  └────────┘  │  │  └────────┘  │  │  └────────┘  │         │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘         │   │
│   │                                                                 │   │
│   │   Each replica on different node = high availability           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Topology Key

The `topologyKey` defines the "domain" for affinity:

| topologyKey | Meaning |
|-------------|---------|
| `kubernetes.io/hostname` | Same node |
| `topology.kubernetes.io/zone` | Same availability zone |
| `topology.kubernetes.io/region` | Same region |

### 4.3 Pod Affinity Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: backend
        topologyKey: kubernetes.io/hostname   # Same node
  containers:
  - name: frontend
    image: nginx
```

### 4.4 Pod Anti-Affinity Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          # MUST not be on same node as other web pods
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx
```

### 4.5 Preferred Pod Anti-Affinity

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: web
        topologyKey: topology.kubernetes.io/zone
```

### 4.6 Combined Affinity and Anti-Affinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      affinity:
        # Run near backend pods
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: backend
              topologyKey: topology.kubernetes.io/zone
        # Don't run on same node as other frontend pods
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web-frontend
            topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx
```

---

## 5. Taints and Tolerations

### 5.1 What are Taints and Tolerations?

**Taints** are applied to nodes to repel pods. **Tolerations** are applied to pods to allow scheduling on tainted nodes.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TAINTS AND TOLERATIONS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   TAINT on Node: "I repel pods unless they tolerate me"                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌──────────────────────────────────┐                          │   │
│   │   │         Node (GPU)               │                          │   │
│   │   │                                  │                          │   │
│   │   │   Taint: gpu=true:NoSchedule     │                          │   │
│   │   │                                  │                          │   │
│   │   │   ┌─────┐  ❌  Regular pods      │                          │   │
│   │   │   │ Pod │────▶ can't schedule   │                          │   │
│   │   │   └─────┘      here             │                          │   │
│   │   │                                  │                          │   │
│   │   │   ┌─────┐  ✅  Pod with         │                          │   │
│   │   │   │ GPU │────▶ toleration       │                          │   │
│   │   │   │ Pod │      CAN schedule     │                          │   │
│   │   │   └─────┘                        │                          │   │
│   │   │                                  │                          │   │
│   │   └──────────────────────────────────┘                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Use Cases:                                                            │
│   • Dedicated nodes (GPU, high-memory)                                 │
│   • Master nodes (no workloads)                                        │
│   • Node maintenance                                                    │
│   • Special hardware (FPGA, TPU)                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Taint Effects

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New pods won't be scheduled (existing stay) |
| `PreferNoSchedule` | Try to avoid scheduling (soft) |
| `NoExecute` | Evict existing pods + block new ones |

### 5.3 Taint Commands

```bash
# Add taint to node
kubectl taint nodes node1 key=value:NoSchedule

# Examples
kubectl taint nodes node1 gpu=true:NoSchedule
kubectl taint nodes node1 dedicated=ml:NoExecute
kubectl taint nodes node1 maintenance=true:NoSchedule

# Remove taint (add minus at end)
kubectl taint nodes node1 gpu=true:NoSchedule-

# View taints on node
kubectl describe node node1 | grep Taints
```

### 5.4 Toleration in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: gpu-app
    image: tensorflow/tensorflow:latest-gpu
```

### 5.5 Toleration Operators

| Operator | Behavior |
|----------|----------|
| `Equal` | Key and value must match |
| `Exists` | Key exists (any value) |

```yaml
# Equal operator - matches gpu=true
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"

# Exists operator - matches any value for gpu key
tolerations:
- key: "gpu"
  operator: "Exists"
  effect: "NoSchedule"

# Tolerate ALL taints (use with caution!)
tolerations:
- operator: "Exists"
```

### 5.6 NoExecute with tolerationSeconds

```yaml
# Pod will be evicted after 600 seconds on NoExecute taint
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 600
```

### 5.7 Built-in Taints

| Taint | When Applied |
|-------|--------------|
| `node.kubernetes.io/not-ready` | Node not ready |
| `node.kubernetes.io/unreachable` | Node unreachable |
| `node.kubernetes.io/memory-pressure` | Node has memory pressure |
| `node.kubernetes.io/disk-pressure` | Node has disk pressure |
| `node.kubernetes.io/pid-pressure` | Node has PID pressure |
| `node.kubernetes.io/unschedulable` | Node is cordoned |

---

## 6. Pod Topology Spread Constraints

### 6.1 What is Topology Spread?

Distributes pods evenly across topology domains (zones, nodes, regions).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOPOLOGY SPREAD                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   WITHOUT Topology Spread:                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Zone A                         Zone B                         │   │
│   │   ┌─────────────────────┐       ┌─────────────────────┐        │   │
│   │   │ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ │       │                     │        │   │
│   │   │ │P│ │P│ │P│ │P│ │P│ │       │      (empty)        │        │   │
│   │   │ └─┘ └─┘ └─┘ └─┘ └─┘ │       │                     │        │   │
│   │   └─────────────────────┘       └─────────────────────┘        │   │
│   │                                                                 │   │
│   │   Zone A fails → ALL pods lost!                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   WITH Topology Spread (maxSkew: 1):                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Zone A                         Zone B                         │   │
│   │   ┌─────────────────────┐       ┌─────────────────────┐        │   │
│   │   │    ┌─┐ ┌─┐ ┌─┐      │       │    ┌─┐ ┌─┐          │        │   │
│   │   │    │P│ │P│ │P│      │       │    │P│ │P│          │        │   │
│   │   │    └─┘ └─┘ └─┘      │       │    └─┘ └─┘          │        │   │
│   │   └─────────────────────┘       └─────────────────────┘        │   │
│   │                                                                 │   │
│   │   Zone A fails → 2 pods survive in Zone B!                     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Topology Spread YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-deployment
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                          # Max difference between zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule    # or ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: web
        image: nginx
```

### 6.3 Topology Spread Parameters

| Parameter | Description |
|-----------|-------------|
| `maxSkew` | Maximum allowed difference in pod count |
| `topologyKey` | Node label defining topology domain |
| `whenUnsatisfiable` | `DoNotSchedule` or `ScheduleAnyway` |
| `labelSelector` | Which pods to consider for spread |

### 6.4 Multiple Spread Constraints

```yaml
topologySpreadConstraints:
# Spread across zones
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: web
# Also spread across nodes within zones
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: web
```

---

## 7. Manual Scheduling

### 7.1 nodeName

Bypass the scheduler entirely by specifying the exact node:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: node1              # Directly assign to node1
  containers:
  - name: app
    image: nginx
```

**⚠️ Warnings:**
- Scheduler is bypassed completely
- No resource checking
- No affinity/taint checking
- Pod will be Pending if node doesn't exist

### 7.2 Static Pods

Pods managed directly by kubelet (not API server):

```bash
# Place manifest in static pod path (usually /etc/kubernetes/manifests/)
/etc/kubernetes/manifests/my-static-pod.yaml
```

---

## 8. Priority and Preemption

### 8.1 What is Priority?

Higher priority pods can preempt (evict) lower priority pods when resources are scarce.

### 8.2 PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000                  # Higher = more important
globalDefault: false            # Is this the default?
preemptionPolicy: PreemptLowerPriority   # or Never
description: "Critical workloads"
```

### 8.3 Using PriorityClass in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```

### 8.4 Built-in Priority Classes

| Class | Value | Use |
|-------|-------|-----|
| `system-node-critical` | 2000001000 | Node-critical pods |
| `system-cluster-critical` | 2000000000 | Cluster-critical pods |

---

## 9. Command Reference

### Node Commands

```bash
# View nodes with labels
kubectl get nodes --show-labels

# Label node
kubectl label nodes node1 disktype=ssd

# Remove label
kubectl label nodes node1 disktype-

# Taint node
kubectl taint nodes node1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-

# View node taints
kubectl describe node node1 | grep Taints

# Cordon node (mark unschedulable)
kubectl cordon node1

# Uncordon node
kubectl uncordon node1

# Drain node (evict pods)
kubectl drain node1 --ignore-daemonsets
```

### Pod Scheduling Commands

```bash
# View pod's node
kubectl get pod <pod> -o wide

# View scheduling events
kubectl describe pod <pod> | grep -A 10 Events

# View scheduler logs
kubectl logs -n kube-system kube-scheduler-<master>

# See why pod is pending
kubectl describe pod <pod> | grep -A 5 "Events"
```

---

## 10. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| nodeSelector | 🔴 HIGH | Simple node selection |
| Node Affinity | 🔴 HIGH | required/preferred |
| Taints/Tolerations | 🔴 HIGH | Create taints, tolerations |
| Pod Anti-Affinity | 🟡 MEDIUM | HA deployment |
| Cordon/Drain | 🟡 MEDIUM | Node maintenance |

### Quick Reference for Exam

```yaml
# nodeSelector
spec:
  nodeSelector:
    disktype: ssd

# Node Affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: [ssd]

# Toleration
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Schedule pod only on nodes with SSD"                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  spec:                                                          │   │
│   │    nodeSelector:                                                │   │
│   │      disktype: ssd                                              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Taint node1 to not accept any pods"                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl taint nodes node1 dedicated=special:NoSchedule         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Make pod tolerate the taint"                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  tolerations:                                                   │   │
│   │  - key: "dedicated"                                             │   │
│   │    operator: "Equal"                                            │   │
│   │    value: "special"                                             │   │
│   │    effect: "NoSchedule"                                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 4: "Drain node for maintenance"                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl drain node1 --ignore-daemonsets --force               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEDULING DECISION TREE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Need to control where pods run?                                       │
│       │                                                                 │
│       ├──▶ Simple label match? ──▶ nodeSelector                        │
│       │                                                                 │
│       ├──▶ Complex node requirements? ──▶ Node Affinity                │
│       │       ├── Must have ──▶ requiredDuring...                      │
│       │       └── Prefer ──▶ preferredDuring...                        │
│       │                                                                 │
│       ├──▶ Based on other pods? ──▶ Pod Affinity/Anti-Affinity         │
│       │       ├── Together ──▶ podAffinity                             │
│       │       └── Apart ──▶ podAntiAffinity                            │
│       │                                                                 │
│       ├──▶ Dedicated nodes? ──▶ Taints + Tolerations                   │
│       │                                                                 │
│       ├──▶ Even distribution? ──▶ Topology Spread                      │
│       │                                                                 │
│       └──▶ Specific node? ──▶ nodeName (bypass scheduler)              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Health Probes** - Liveness, readiness, and startup probes
- Probe types (HTTP, TCP, exec)
- Probe configuration
- Best practices

---

**Chapter 22 Complete! 🎉**

You now understand:
- How the scheduler works
- nodeSelector for simple node selection
- Node Affinity for advanced rules
- Pod Affinity/Anti-Affinity for pod-based placement
- Taints and Tolerations for node restrictions
- Topology Spread for even distribution
- Manual scheduling with nodeName
- Priority and preemption
- CKA exam preparation

