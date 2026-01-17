# Chapter 21: Resource Management - CPU, Memory, and QoS

## Introduction

In a shared cluster, resource management is critical. Without proper limits, one runaway pod can consume all available resources, starving other applications. Kubernetes provides powerful mechanisms to manage, allocate, and enforce resource usage.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 THE RESOURCE MANAGEMENT PROBLEM                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Resource Management:                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Node Capacity: 4 CPU, 8GB Memory                              │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │ ████████████████████████████████████████████████████████│   │   │
│   │   │                    Pod A (Runaway)                      │   │   │
│   │   │                    Uses: 3.8 CPU, 7.5GB                 │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │   ┌──┐ ┌──┐ ┌──┐                                                │   │
│   │   │B │ │C │ │D │ ← Starved! OOMKilled! CPU throttled!          │   │
│   │   └──┘ └──┘ └──┘                                                │   │
│   │                                                                 │   │
│   │   Result: Other pods fail, node becomes unstable               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Resource Management:                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Node Capacity: 4 CPU, 8GB Memory                              │   │
│   │   ┌───────────────┐ ┌───────────────┐                          │   │
│   │   │    Pod A      │ │    Pod B      │                          │   │
│   │   │ Limit: 1 CPU  │ │ Limit: 1 CPU  │                          │   │
│   │   │       2GB     │ │       2GB     │                          │   │
│   │   └───────────────┘ └───────────────┘                          │   │
│   │   ┌───────────────┐ ┌───────────────┐                          │   │
│   │   │    Pod C      │ │    Pod D      │                          │   │
│   │   │ Limit: 1 CPU  │ │ Limit: 1 CPU  │                          │   │
│   │   │       2GB     │ │       2GB     │                          │   │
│   │   └───────────────┘ └───────────────┘                          │   │
│   │                                                                 │   │
│   │   Result: Fair sharing, predictable performance                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Resource Types

### 1.1 Compressible vs Incompressible Resources

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     RESOURCE TYPES                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   COMPRESSIBLE (CPU)                    INCOMPRESSIBLE (Memory)         │
│   ┌─────────────────────────────┐       ┌─────────────────────────────┐ │
│   │                             │       │                             │ │
│   │  Can be throttled           │       │  Cannot be compressed       │ │
│   │                             │       │                             │ │
│   │  If container exceeds       │       │  If container exceeds       │ │
│   │  CPU limit:                 │       │  memory limit:              │ │
│   │  → CPU is THROTTLED         │       │  → Container is KILLED      │ │
│   │  → Container keeps running  │       │  → OOMKilled                │ │
│   │  → Performance degrades     │       │  → Must restart             │ │
│   │                             │       │                             │ │
│   │  Analogy: Bandwidth limit   │       │  Analogy: Storage limit     │ │
│   │  (slower but works)         │       │  (exceeds = crash)          │ │
│   │                             │       │                             │ │
│   └─────────────────────────────┘       └─────────────────────────────┘ │
│                                                                         │
│   Other Resource Types:                                                 │
│   • Ephemeral Storage: Like memory (incompressible)                    │
│   • Extended Resources: GPUs, FPGAs (custom)                           │
│   • Hugepages: Large memory pages                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 CPU Units

| Format | Description | Example |
|--------|-------------|---------|
| Cores | Whole CPU cores | `1`, `2`, `4` |
| Millicores | 1/1000 of a core | `100m`, `500m`, `1500m` |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CPU UNITS                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1 CPU = 1000 millicores (m)                                           │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                        1 CPU Core                               │   │
│   │  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐                               │   │
│   │  │  │  │  │  │  │  │  │  │  │  │  = 1000m                      │   │
│   │  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘                               │   │
│   │   100m each                                                     │   │
│   │                                                                 │   │
│   │   100m = 0.1 CPU = 10% of one core                             │   │
│   │   500m = 0.5 CPU = 50% of one core                             │   │
│   │   1000m = 1 CPU = 100% of one core                             │   │
│   │   1500m = 1.5 CPU = 150% (uses parts of 2 cores)               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Platform Equivalents:                                                 │
│   • AWS: 1 vCPU = 1 CPU                                                │
│   • GCP: 1 vCPU = 1 CPU                                                │
│   • Azure: 1 vCore = 1 CPU                                             │
│   • Bare metal: 1 hyperthread = 1 CPU                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Memory Units

| Format | Description | Size |
|--------|-------------|------|
| Bytes | Plain number | `134217728` |
| Kilobytes | K (1000) or Ki (1024) | `128K`, `128Ki` |
| Megabytes | M (1000²) or Mi (1024²) | `128M`, `128Mi` |
| Gigabytes | G (1000³) or Gi (1024³) | `1G`, `1Gi` |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       MEMORY UNITS                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Decimal (SI) vs Binary (IEC):                                         │
│                                                                         │
│   ┌─────────────────────────────┬─────────────────────────────┐        │
│   │     Decimal (SI)            │      Binary (IEC)           │        │
│   ├─────────────────────────────┼─────────────────────────────┤        │
│   │  K  = 1000 bytes            │  Ki = 1024 bytes            │        │
│   │  M  = 1000² = 1,000,000     │  Mi = 1024² = 1,048,576     │        │
│   │  G  = 1000³ = 1,000,000,000 │  Gi = 1024³ = 1,073,741,824 │        │
│   └─────────────────────────────┴─────────────────────────────┘        │
│                                                                         │
│   Common Examples:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  128Mi  = 128 × 1024 × 1024 = 134,217,728 bytes ≈ 134 MB       │   │
│   │  256Mi  = 268,435,456 bytes ≈ 268 MB                           │   │
│   │  512Mi  = 536,870,912 bytes ≈ 537 MB                           │   │
│   │  1Gi    = 1,073,741,824 bytes ≈ 1.07 GB                        │   │
│   │  2Gi    = 2,147,483,648 bytes ≈ 2.15 GB                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ⚠️  Use binary units (Mi, Gi) for consistency with OS reporting      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Requests and Limits

### 2.1 Understanding Requests vs Limits

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  REQUESTS vs LIMITS                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   REQUEST = "I need at least this much"                                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Used for SCHEDULING decisions                               │   │
│   │  • Scheduler ensures node has enough capacity                  │   │
│   │  • Guaranteed allocation                                       │   │
│   │  • Pod won't be scheduled if request can't be met              │   │
│   │                                                                 │   │
│   │  Think: Reservation at a restaurant                            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   LIMIT = "I cannot exceed this much"                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Used for ENFORCEMENT at runtime                             │   │
│   │  • Container cannot exceed this amount                         │   │
│   │  • CPU: Throttled if exceeded                                  │   │
│   │  • Memory: OOMKilled if exceeded                               │   │
│   │                                                                 │   │
│   │  Think: Credit card spending limit                             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Visual:                                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   0         Request           Limit          Node Capacity     │   │
│   │   ├────────────┼────────────────┼────────────────┤             │   │
│   │   │            │                │                │             │   │
│   │   │◀──────────▶│◀──────────────▶│                │             │   │
│   │   │ Guaranteed │  Burstable     │  FORBIDDEN     │             │   │
│   │   │            │  (when avail)  │  (throttle/OOM)│             │   │
│   │   │            │                │                │             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Setting Requests and Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:              # Minimum guaranteed
        cpu: "100m"          # 0.1 CPU
        memory: "128Mi"      # 128 MiB
      limits:                # Maximum allowed
        cpu: "500m"          # 0.5 CPU
        memory: "256Mi"      # 256 MiB
```

### 2.3 What Happens When Limits Are Exceeded

| Resource | Behavior When Exceeded |
|----------|----------------------|
| **CPU** | Container is **throttled** (slowed down), keeps running |
| **Memory** | Container is **OOMKilled** (terminated), must restart |
| **Ephemeral Storage** | Pod is **evicted** from node |

```
┌─────────────────────────────────────────────────────────────────────────┐
│               WHAT HAPPENS WHEN LIMITS ARE EXCEEDED                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CPU LIMIT EXCEEDED:                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Container requests 800m CPU, Limit is 500m                    │   │
│   │                                                                 │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  CPU Scheduler                                          │   │   │
│   │   │                                                         │   │   │
│   │   │  "Container wants 800m but limit is 500m"               │   │   │
│   │   │  "Throttling CPU to 500m"                               │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   Result: Container runs slower, but keeps running             │   │
│   │   Symptom: High CPU throttling in metrics                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   MEMORY LIMIT EXCEEDED:                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Container tries to allocate 300Mi, Limit is 256Mi             │   │
│   │                                                                 │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  Linux OOM Killer                                       │   │   │
│   │   │                                                         │   │   │
│   │   │  "Container exceeded memory limit!"                     │   │   │
│   │   │  "Killing process with signal SIGKILL"                  │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   Result: Container terminates with exit code 137 (OOMKilled)  │   │
│   │   Pod Status: OOMKilled, may restart based on restartPolicy    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Request/Limit Combinations

| Requests | Limits | Behavior |
|----------|--------|----------|
| ✅ Set | ✅ Set | Best practice - guaranteed baseline, capped max |
| ✅ Set | ❌ Not set | Can use all available node resources |
| ❌ Not set | ✅ Set | Request defaults to limit value |
| ❌ Not set | ❌ Not set | No guarantees, lowest scheduling priority |

---

## 3. Quality of Service (QoS) Classes

### 3.1 QoS Classes Overview

Kubernetes assigns one of three QoS classes to each pod based on resource specifications:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    QoS CLASSES                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     GUARANTEED                                  │   │
│   │                                                                 │   │
│   │   Criteria:                                                     │   │
│   │   • Every container has CPU and memory requests AND limits     │   │
│   │   • Requests EQUAL limits for both CPU and memory              │   │
│   │                                                                 │   │
│   │   Priority: HIGHEST                                             │   │
│   │   • Last to be evicted under memory pressure                   │   │
│   │   • Most predictable performance                               │   │
│   │   • Use for: Databases, critical services                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      BURSTABLE                                  │   │
│   │                                                                 │   │
│   │   Criteria:                                                     │   │
│   │   • At least one container has request OR limit                │   │
│   │   • Does not meet Guaranteed criteria                          │   │
│   │                                                                 │   │
│   │   Priority: MEDIUM                                              │   │
│   │   • Can burst above requests if resources available            │   │
│   │   • Evicted before Guaranteed pods                             │   │
│   │   • Use for: Web servers, general workloads                    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      BESTEFFORT                                 │   │
│   │                                                                 │   │
│   │   Criteria:                                                     │   │
│   │   • No containers have requests or limits set                  │   │
│   │                                                                 │   │
│   │   Priority: LOWEST                                              │   │
│   │   • First to be evicted under resource pressure                │   │
│   │   • Uses only spare resources                                  │   │
│   │   • Use for: Batch jobs, non-critical tasks                    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 QoS Class Examples

#### Guaranteed QoS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"          # MUST equal request
        memory: "256Mi"      # MUST equal request
```

#### Burstable QoS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"          # Different from request
        memory: "256Mi"      # Different from request
```

#### BestEffort QoS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx
    # No resources specified at all
```

### 3.3 Check Pod QoS Class

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# Or in describe
kubectl describe pod <pod-name> | grep "QoS Class"
```

---

## 4. LimitRange

### 4.1 What is LimitRange?

A **LimitRange** sets default requests/limits and min/max constraints for a namespace. It's applied when pods don't specify their own resources.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      LIMITRANGE OVERVIEW                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     Namespace: dev                              │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                    LimitRange                           │   │   │
│   │   │                                                         │   │   │
│   │   │  Container Defaults:                                    │   │   │
│   │   │    defaultRequest: cpu=100m, memory=128Mi               │   │   │
│   │   │    default: cpu=500m, memory=256Mi                      │   │   │
│   │   │                                                         │   │   │
│   │   │  Container Constraints:                                 │   │   │
│   │   │    min: cpu=50m, memory=64Mi                           │   │   │
│   │   │    max: cpu=2, memory=2Gi                              │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   Pod without resources:                                        │   │
│   │   ┌─────────────────────┐                                       │   │
│   │   │  containers:        │   Gets defaults applied:              │   │
│   │   │  - name: app        │   → request: cpu=100m, memory=128Mi   │   │
│   │   │    image: nginx     │   → limit: cpu=500m, memory=256Mi     │   │
│   │   │    # no resources   │                                       │   │
│   │   └─────────────────────┘                                       │   │
│   │                                                                 │   │
│   │   Pod with resources > max:                                     │   │
│   │   ┌─────────────────────┐                                       │   │
│   │   │  resources:         │   REJECTED!                           │   │
│   │   │    limits:          │   "cpu max is 2 but spec is 4"       │   │
│   │   │      cpu: "4"       │                                       │   │
│   │   └─────────────────────┘                                       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 LimitRange YAML

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: dev
spec:
  limits:
  # Container-level limits
  - type: Container
    default:              # Default limits if not specified
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:       # Default requests if not specified
      cpu: "100m"
      memory: "128Mi"
    min:                  # Minimum allowed
      cpu: "50m"
      memory: "64Mi"
    max:                  # Maximum allowed
      cpu: "2"
      memory: "2Gi"
    maxLimitRequestRatio: # Limit/Request ratio
      cpu: "10"
      memory: "4"
  
  # Pod-level limits
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
  
  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: "1Gi"
    max:
      storage: "10Gi"
```

### 4.3 LimitRange Fields

| Field | Description |
|-------|-------------|
| `default` | Default limits applied to containers |
| `defaultRequest` | Default requests applied to containers |
| `min` | Minimum resource values allowed |
| `max` | Maximum resource values allowed |
| `maxLimitRequestRatio` | Maximum ratio of limit/request |
| `type` | Container, Pod, or PersistentVolumeClaim |

### 4.4 LimitRange Commands

```bash
# Create LimitRange
kubectl apply -f limitrange.yaml

# List LimitRanges
kubectl get limitrange -n dev

# Describe LimitRange
kubectl describe limitrange resource-limits -n dev

# Delete LimitRange
kubectl delete limitrange resource-limits -n dev
```

---

## 5. ResourceQuota

### 5.1 What is ResourceQuota?

A **ResourceQuota** limits the total amount of resources that can be consumed in a namespace.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     RESOURCEQUOTA OVERVIEW                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     Namespace: team-a                           │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                   ResourceQuota                         │   │   │
│   │   │                                                         │   │   │
│   │   │  Compute:                                               │   │   │
│   │   │    requests.cpu: 10                                     │   │   │
│   │   │    requests.memory: 20Gi                                │   │   │
│   │   │    limits.cpu: 20                                       │   │   │
│   │   │    limits.memory: 40Gi                                  │   │   │
│   │   │                                                         │   │   │
│   │   │  Object Count:                                          │   │   │
│   │   │    pods: 20                                             │   │   │
│   │   │    services: 10                                         │   │   │
│   │   │    secrets: 50                                          │   │   │
│   │   │                                                         │   │   │
│   │   │  Storage:                                               │   │   │
│   │   │    requests.storage: 100Gi                              │   │   │
│   │   │    persistentvolumeclaims: 10                           │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   Current Usage:                                                │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  requests.cpu:    5/10  (50%)   ████████░░░░░░░░░░     │   │   │
│   │   │  requests.memory: 15/20Gi (75%) ████████████████░░░░   │   │   │
│   │   │  pods:           15/20  (75%)   ████████████████░░░░   │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   New pod requesting 6 CPU? REJECTED! (would exceed quota)     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 ResourceQuota YAML

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    # Compute Resources
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    
    # Object Count
    pods: "20"
    replicationcontrollers: "10"
    services: "10"
    services.loadbalancers: "2"
    services.nodeports: "5"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "10"
    
    # Storage
    requests.storage: "100Gi"
    
    # Per-StorageClass storage
    gold.storageclass.storage.k8s.io/requests.storage: "50Gi"
    gold.storageclass.storage.k8s.io/persistentvolumeclaims: "5"
```

### 5.3 ResourceQuota with Scopes

Limit quotas to specific pod types:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: dev
spec:
  hard:
    pods: "5"
  scopes:
  - BestEffort          # Only applies to BestEffort pods
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: not-besteffort-quota
  namespace: dev
spec:
  hard:
    pods: "20"
    requests.cpu: "10"
    requests.memory: "20Gi"
  scopes:
  - NotBestEffort       # Applies to Guaranteed and Burstable
```

### 5.4 Quota Scopes

| Scope | Matches Pods That... |
|-------|----------------------|
| `BestEffort` | Have QoS class BestEffort |
| `NotBestEffort` | Have QoS class Guaranteed or Burstable |
| `Terminating` | Have activeDeadlineSeconds set |
| `NotTerminating` | Don't have activeDeadlineSeconds |
| `PriorityClass` | Reference specified priority class |

### 5.5 ResourceQuota Commands

```bash
# Create quota
kubectl apply -f resourcequota.yaml

# List quotas
kubectl get resourcequota -n team-a

# Check quota usage
kubectl describe resourcequota team-quota -n team-a

# Get usage in YAML
kubectl get resourcequota team-quota -n team-a -o yaml
```

### 5.6 Quota Usage Output

```
Name:                   team-quota
Namespace:              team-a
Resource                Used    Hard
--------                ----    ----
configmaps              3       50
limits.cpu              4       20
limits.memory           8Gi     40Gi
persistentvolumeclaims  2       10
pods                    8       20
requests.cpu            2       10
requests.memory         4Gi     20Gi
requests.storage        20Gi    100Gi
secrets                 5       50
services                3       10
```

---

## 6. Node Resource Management

### 6.1 Node Allocatable Resources

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NODE RESOURCE ALLOCATION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    NODE CAPACITY                                │   │
│   │                                                                 │   │
│   │   Total: 16 CPU, 64Gi Memory                                    │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                                                         │   │   │
│   │   │  ┌─────────────────────────────────────────────────┐   │   │   │
│   │   │  │  kube-reserved (for kubelet, runtime)          │   │   │   │
│   │   │  │  CPU: 1, Memory: 2Gi                           │   │   │   │
│   │   │  └─────────────────────────────────────────────────┘   │   │   │
│   │   │  ┌─────────────────────────────────────────────────┐   │   │   │
│   │   │  │  system-reserved (for OS processes)            │   │   │   │
│   │   │  │  CPU: 0.5, Memory: 1Gi                         │   │   │   │
│   │   │  └─────────────────────────────────────────────────┘   │   │   │
│   │   │  ┌─────────────────────────────────────────────────┐   │   │   │
│   │   │  │  eviction-threshold (buffer before eviction)   │   │   │   │
│   │   │  │  Memory: 100Mi                                 │   │   │   │
│   │   │  └─────────────────────────────────────────────────┘   │   │   │
│   │   │  ┌─────────────────────────────────────────────────┐   │   │   │
│   │   │  │                                                 │   │   │   │
│   │   │  │            ALLOCATABLE                          │   │   │   │
│   │   │  │                                                 │   │   │   │
│   │   │  │   CPU: 14.5, Memory: ~60Gi                     │   │   │   │
│   │   │  │                                                 │   │   │   │
│   │   │  │   Available for Pods                           │   │   │   │
│   │   │  │                                                 │   │   │   │
│   │   │  └─────────────────────────────────────────────────┘   │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Formula: Allocatable = Capacity - kube-reserved - system-reserved    │
│                          - eviction-threshold                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 View Node Resources

```bash
# View node capacity and allocatable
kubectl describe node <node-name> | grep -A 10 "Capacity\|Allocatable"

# View node resource usage
kubectl top node

# View pods on a node and their resources
kubectl describe node <node-name> | grep -A 50 "Non-terminated Pods"
```

### 6.3 Node Resource Output

```
Capacity:
  cpu:                16
  ephemeral-storage:  100Gi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             65536Mi
  pods:               110
Allocatable:
  cpu:                15500m
  ephemeral-storage:  95Gi
  memory:             63488Mi
  pods:               110
```

---

## 7. Pod Eviction

### 7.1 Eviction Triggers

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      POD EVICTION                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Eviction Triggers:                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  Memory Pressure                                                │   │
│   │  ├── memory.available < eviction threshold                     │   │
│   │  └── nodefs.available < threshold                              │   │
│   │                                                                 │   │
│   │  Disk Pressure                                                  │   │
│   │  ├── nodefs.available < threshold                              │   │
│   │  └── imagefs.available < threshold                             │   │
│   │                                                                 │   │
│   │  PID Pressure                                                   │   │
│   │  └── pid.available < threshold                                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Eviction Order (Priority):                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  1. BestEffort pods using more than request                    │   │
│   │  2. Burstable pods using more than request                     │   │
│   │  3. Guaranteed pods (only if above their limits)               │   │
│   │                                                                 │   │
│   │  Within same QoS: Pods using most resources evicted first      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Default Eviction Thresholds

| Signal | Default Threshold |
|--------|-------------------|
| `memory.available` | 100Mi |
| `nodefs.available` | 10% |
| `nodefs.inodesFree` | 5% |
| `imagefs.available` | 15% |

---

## 8. Best Practices

### 8.1 Resource Configuration Guidelines

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   RESOURCE BEST PRACTICES                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Always set both requests AND limits                         │   │
│   │  • Use binary units (Mi, Gi) for consistency                   │   │
│   │  • Profile your application to determine real needs            │   │
│   │  • Use LimitRange for namespace defaults                       │   │
│   │  • Use ResourceQuota for namespace capacity                    │   │
│   │  • Monitor actual usage vs requests/limits                     │   │
│   │  • Set requests = limits for Guaranteed QoS (critical apps)    │   │
│   │  • Right-size based on metrics, not guesses                    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Deploy without any resource specs (BestEffort)              │   │
│   │  • Set requests much higher than actual usage                  │   │
│   │  • Set limits too close to requests (no room to burst)        │   │
│   │  • Ignore OOMKilled events (memory leak indicator)             │   │
│   │  • Over-commit cluster beyond safe limits                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Sizing Guidelines

| Workload Type | Request | Limit | QoS |
|---------------|---------|-------|-----|
| **Critical Database** | Actual need | = Request | Guaranteed |
| **Web Server** | Average usage | 2-3x request | Burstable |
| **Batch Job** | Average usage | Node capacity | Burstable |
| **Dev/Test** | Low baseline | Higher limit | Burstable |

---

## 9. Command Reference

### View Resource Usage

```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods
kubectl top pods -n <namespace>
kubectl top pods --containers   # Per-container

# Pod resource specs
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'

# Check pod QoS
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```

### LimitRange Commands

```bash
# Create
kubectl apply -f limitrange.yaml

# List
kubectl get limitrange -n <namespace>

# Describe
kubectl describe limitrange <name> -n <namespace>

# Delete
kubectl delete limitrange <name> -n <namespace>
```

### ResourceQuota Commands

```bash
# Create
kubectl apply -f resourcequota.yaml

# List
kubectl get resourcequota -n <namespace>

# Check usage
kubectl describe resourcequota <name> -n <namespace>

# Delete
kubectl delete resourcequota <name> -n <namespace>
```

---

## 10. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Set requests/limits | 🔴 HIGH | Add to pod spec |
| LimitRange | 🔴 HIGH | Create defaults |
| ResourceQuota | 🔴 HIGH | Set namespace limits |
| QoS Classes | 🟡 MEDIUM | Understand criteria |
| Monitor usage | 🟡 MEDIUM | kubectl top |

### Quick Reference for Exam

```yaml
# Container resources
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```yaml
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

```yaml
# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "20"
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Add resources to a pod"                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  containers:                                                    │   │
│   │  - name: app                                                    │   │
│   │    resources:                                                   │   │
│   │      requests:                                                  │   │
│   │        cpu: "100m"                                              │   │
│   │        memory: "128Mi"                                          │   │
│   │      limits:                                                    │   │
│   │        cpu: "200m"                                              │   │
│   │        memory: "256Mi"                                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Create LimitRange with defaults"                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl create -f - <<EOF                                      │   │
│   │  apiVersion: v1                                                 │   │
│   │  kind: LimitRange                                               │   │
│   │  metadata:                                                      │   │
│   │    name: default-limits                                         │   │
│   │    namespace: dev                                               │   │
│   │  spec:                                                          │   │
│   │    limits:                                                      │   │
│   │    - type: Container                                            │   │
│   │      default:                                                   │   │
│   │        cpu: "500m"                                              │   │
│   │  EOF                                                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Why is this pod pending?"                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe pod <name>                                    │   │
│   │  Look for: "Insufficient cpu" or "Insufficient memory"         │   │
│   │  Check: Node capacity vs requested resources                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Docker to Kubernetes Mapping

| Docker Flag | Kubernetes Equivalent |
|-------------|----------------------|
| `--memory=256m` | `resources.limits.memory: 256Mi` |
| `--memory-reservation=128m` | `resources.requests.memory: 128Mi` |
| `--cpus=0.5` | `resources.limits.cpu: 500m` |
| `--cpu-shares=512` | `resources.requests.cpu` (relative) |
| `--oom-kill-disable` | No equivalent (OOM handling is automatic) |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  RESOURCE MANAGEMENT SUMMARY                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CONTAINER LEVEL:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  resources:                                                     │   │
│   │    requests: (for scheduling)                                   │   │
│   │    limits: (for enforcement)                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   NAMESPACE LEVEL:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  LimitRange: Default/min/max per container or pod              │   │
│   │  ResourceQuota: Total limits for namespace                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   QoS CLASSES:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Guaranteed: requests = limits (highest priority)              │   │
│   │  Burstable: requests < limits (medium priority)                │   │
│   │  BestEffort: no resources (lowest priority)                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Scheduling** - How pods get placed on nodes
- nodeSelector, node affinity, pod affinity
- Taints and tolerations
- Pod topology spread constraints

---

**Chapter 21 Complete! 🎉**

You now understand:
- CPU and memory units
- Requests vs limits
- Quality of Service (QoS) classes
- LimitRange for namespace defaults
- ResourceQuota for namespace limits
- Node resource management
- Pod eviction
- Best practices
- CKA exam preparation

