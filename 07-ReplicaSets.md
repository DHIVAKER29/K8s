# ☸️ Chapter 07: ReplicaSets

> Understanding ReplicaSets - the controller that ensures a specified number of pod replicas are running at all times.

---

## 📚 Table of Contents

1. [What is a ReplicaSet?](#-what-is-a-replicaset)
2. [Why ReplicaSets?](#-why-replicasets)
3. [ReplicaSet Architecture](#-replicaset-architecture)
4. [ReplicaSet Manifest](#-replicaset-manifest)
5. [Creating ReplicaSets](#-creating-replicasets)
6. [Selectors Deep Dive](#-selectors-deep-dive)
7. [Scaling ReplicaSets](#-scaling-replicasets)
8. [Pod Template](#-pod-template)
9. [How ReplicaSets Work](#-how-replicasets-work)
10. [ReplicaSet vs Deployment](#-replicaset-vs-deployment)
11. [Ownership and Labels](#-ownership-and-labels)
12. [Common Operations](#-common-operations)
13. [Troubleshooting](#-troubleshooting)
14. [CKA Exam Tips](#-cka-exam-tips)
15. [Summary](#-summary)

---

## 📖 What is a ReplicaSet?

### Definition

> **ReplicaSet** is a Kubernetes controller that maintains a stable set of replica Pods running at any given time. It guarantees the availability of a specified number of identical Pods.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS A REPLICASET?                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  "I want exactly 3 copies of this pod running at all times"                         │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │                         ReplicaSet: nginx-rs                                   ││
│  │                         replicas: 3                                            ││
│  │                                                                                ││
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                ││
│  │  │   Pod: nginx-1  │  │   Pod: nginx-2  │  │   Pod: nginx-3  │                ││
│  │  │   image: nginx  │  │   image: nginx  │  │   image: nginx  │                ││
│  │  │   labels:       │  │   labels:       │  │   labels:       │                ││
│  │  │     app: nginx  │  │     app: nginx  │  │     app: nginx  │                ││
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘                ││
│  │                                                                                ││
│  │  If any pod dies, ReplicaSet creates a new one to maintain count of 3        ││
│  │                                                                                ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  ReplicaSet = Desired Pod Count + Pod Template + Selector                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### ReplicaSet Components

| Component | Purpose |
|-----------|---------|
| **replicas** | Number of pod copies to maintain |
| **selector** | How to find pods it manages |
| **template** | Blueprint for creating new pods |

---

## ❓ Why ReplicaSets?

### The Problem: Pods Are Ephemeral

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         THE PROBLEM WITH BARE PODS                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Without ReplicaSet:                                                                │
│                                                                                      │
│  Time T0: Pod running                    Time T1: Pod crashes                       │
│  ┌─────────────────┐                     ┌─────────────────┐                        │
│  │   Pod: nginx    │                     │   Pod: nginx    │                        │
│  │   Status: ✅    │       ───────►      │   Status: 💀    │                        │
│  │   Running       │       Crash!        │   CrashLoop     │                        │
│  └─────────────────┘                     └─────────────────┘                        │
│                                                                                      │
│  Nobody recreates it! Application is DOWN!                                          │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  With ReplicaSet:                                                                   │
│                                                                                      │
│  Time T0: 3 Pods              Time T1: Pod crashes       Time T2: Auto-healed      │
│  ┌───┐ ┌───┐ ┌───┐            ┌───┐ ┌───┐ ┌───┐         ┌───┐ ┌───┐ ┌───┐         │
│  │ ✅│ │ ✅│ │ ✅│   ────►    │ ✅│ │ ✅│ │ 💀│   ────► │ ✅│ │ ✅│ │ ✅│          │
│  └───┘ └───┘ └───┘            └───┘ └───┘ └───┘         └───┘ └───┘ └───┘         │
│                                                                                      │
│  ReplicaSet notices: "I have 2 pods but want 3" → Creates new pod                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Benefits of ReplicaSets

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         REPLICASET BENEFITS                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. HIGH AVAILABILITY                                                               │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Multiple replicas = no single point of failure                                   │
│  • Automatic pod replacement on failure                                             │
│  • Maintains desired state continuously                                             │
│                                                                                      │
│  2. SCALING                                                                         │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Easy horizontal scaling (add/remove pods)                                       │
│  • Handle increased load                                                            │
│  • kubectl scale rs/nginx --replicas=10                                            │
│                                                                                      │
│  3. LOAD DISTRIBUTION                                                               │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Traffic distributed across replicas                                             │
│  • Better resource utilization                                                     │
│  • Improved response times                                                         │
│                                                                                      │
│  4. DECLARATIVE MANAGEMENT                                                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Specify desired state                                                           │
│  • Kubernetes maintains it automatically                                           │
│  • No manual intervention needed                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ ReplicaSet Architecture

### How ReplicaSets Fit In

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         REPLICASET IN THE BIG PICTURE                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                        Deployment (optional)                                │    │
│  │                                                                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │    │
│  │  │                         ReplicaSet                                    │  │    │
│  │  │                                                                       │  │    │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │  │    │
│  │  │  │     Pod      │  │     Pod      │  │     Pod      │               │  │    │
│  │  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │               │  │    │
│  │  │  │  │Container│  │  │  │Container│  │  │  │Container│  │               │  │    │
│  │  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │               │  │    │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘               │  │    │
│  │  │                                                                       │  │    │
│  │  └──────────────────────────────────────────────────────────────────────┘  │    │
│  │                                                                             │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  Hierarchy: Deployment → ReplicaSet → Pod → Container                              │
│                                                                                      │
│  Note: Usually you use Deployments, which manage ReplicaSets for you               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Control Loop

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         REPLICASET CONTROL LOOP                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                             │  │
│   │    1. Observe          2. Compare           3. Act                         │  │
│   │                                                                             │  │
│   │   "How many pods      "Current vs          "Create or delete              │  │
│   │    exist with my       desired?"            pods to match"                 │  │
│   │    selector?"                                                              │  │
│   │                                                                             │  │
│   │        │                   │                     │                         │  │
│   │        ▼                   ▼                     ▼                         │  │
│   │   ┌─────────┐         ┌─────────┐          ┌─────────┐                    │  │
│   │   │ Current │         │ Desired │          │  Take   │                    │  │
│   │   │    3    │    vs   │    3    │    ──►   │ Action  │                    │  │
│   │   │  pods   │         │ replicas│          │         │                    │  │
│   │   └─────────┘         └─────────┘          └─────────┘                    │  │
│   │                                                                             │  │
│   └─────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                      │
│                              │ Continuous loop                                      │
│                              ▼                                                      │
│                                                                                      │
│   Scenarios:                                                                        │
│   • Current < Desired → Create pods                                                │
│   • Current > Desired → Delete pods                                                │
│   • Current = Desired → Do nothing                                                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 ReplicaSet Manifest

### Complete ReplicaSet YAML

```yaml
apiVersion: apps/v1                  # API version for ReplicaSet
kind: ReplicaSet                     # Resource type
metadata:
  name: nginx-replicaset             # ReplicaSet name
  namespace: default                 # Namespace
  labels:                            # Labels for the ReplicaSet itself
    app: nginx
    tier: frontend
spec:
  # Number of desired pods
  replicas: 3
  
  # How to find pods this ReplicaSet manages
  selector:
    matchLabels:
      app: nginx                     # Must match template labels
    matchExpressions:                # Optional: more complex selection
    - key: tier
      operator: In
      values:
      - frontend
      - web
  
  # Pod template - blueprint for creating pods
  template:
    metadata:
      labels:
        app: nginx                   # MUST match selector
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Minimal ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

---

## 🛠️ Creating ReplicaSets

### Create and Manage

```bash
# Create ReplicaSet
kubectl apply -f replicaset.yaml

# List ReplicaSets
kubectl get replicasets
kubectl get rs              # Short form
kubectl get rs -o wide

# Describe ReplicaSet
kubectl describe rs nginx-rs

# View pods created by ReplicaSet
kubectl get pods -l app=nginx

# Check ReplicaSet events
kubectl describe rs nginx-rs | grep -A 10 "Events"

# Delete ReplicaSet (also deletes pods)
kubectl delete rs nginx-rs

# Delete ReplicaSet but keep pods (orphan)
kubectl delete rs nginx-rs --cascade=orphan
```

### Quick Create (for testing)

```bash
# No direct imperative command for ReplicaSet
# Use dry-run to generate YAML

# Option 1: Create from Deployment template, modify
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml | \
  sed 's/Deployment/ReplicaSet/g' > rs.yaml

# Option 2: Write minimal YAML
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

---

## 🔍 Selectors Deep Dive

### How Selectors Work

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         REPLICASET SELECTORS                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  The selector defines which pods the ReplicaSet manages.                            │
│                                                                                      │
│  ReplicaSet                              Pods in Cluster                            │
│  ┌─────────────────────────┐             ┌──────────────────┐                      │
│  │ selector:               │             │ Pod A            │                      │
│  │   matchLabels:          │────────────►│ labels:          │ ✅ Managed           │
│  │     app: nginx          │             │   app: nginx     │                      │
│  └─────────────────────────┘             └──────────────────┘                      │
│                                          ┌──────────────────┐                      │
│                             ────────────►│ Pod B            │ ✅ Managed           │
│                                          │ labels:          │                      │
│                                          │   app: nginx     │                      │
│                                          │   env: prod      │                      │
│                                          └──────────────────┘                      │
│                                          ┌──────────────────┐                      │
│                             ─────────X   │ Pod C            │ ❌ NOT Managed       │
│                                          │ labels:          │    (different label) │
│                                          │   app: apache    │                      │
│                                          └──────────────────┘                      │
│                                                                                      │
│  Key Rule: Pod labels MUST contain all selector labels (can have more)             │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Selector Types

```yaml
# 1. Equality-based (matchLabels)
spec:
  selector:
    matchLabels:
      app: nginx           # app == nginx
      environment: prod    # AND environment == prod

# 2. Set-based (matchExpressions)
spec:
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - nginx
      - apache
    - key: environment
      operator: NotIn
      values:
      - development
    - key: tier
      operator: Exists
    - key: deprecated
      operator: DoesNotExist

# 3. Combined (both)
spec:
  selector:
    matchLabels:
      app: nginx
    matchExpressions:
    - key: version
      operator: In
      values:
      - v1
      - v2
```

### Selector Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `In` | Value in set | `app In [nginx, apache]` |
| `NotIn` | Value not in set | `env NotIn [dev, test]` |
| `Exists` | Key exists (any value) | `tier Exists` |
| `DoesNotExist` | Key doesn't exist | `deprecated DoesNotExist` |

### ⚠️ Critical Rule: Selector Must Match Template

```yaml
# ❌ WRONG - selector doesn't match template labels
spec:
  selector:
    matchLabels:
      app: nginx           # Looking for app=nginx
  template:
    metadata:
      labels:
        app: web           # But template has app=web!

# ✅ CORRECT - selector matches template labels
spec:
  selector:
    matchLabels:
      app: nginx           # Looking for app=nginx
  template:
    metadata:
      labels:
        app: nginx         # Template has app=nginx ✓
        tier: frontend     # Extra labels are OK
```

---

## 📈 Scaling ReplicaSets

### Scaling Methods

```bash
# Method 1: kubectl scale
kubectl scale rs nginx-rs --replicas=5

# Method 2: kubectl edit (opens editor)
kubectl edit rs nginx-rs
# Change replicas value and save

# Method 3: kubectl patch
kubectl patch rs nginx-rs -p '{"spec":{"replicas":5}}'

# Method 4: Update YAML and apply
# Edit replicaset.yaml, change replicas
kubectl apply -f replicaset.yaml
```

### Scaling Visualization

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SCALING REPLICASETS                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Scale Up: replicas 3 → 5                                                           │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Before: ┌───┐ ┌───┐ ┌───┐                                                          │
│          │ 1 │ │ 2 │ │ 3 │                                                          │
│          └───┘ └───┘ └───┘                                                          │
│                                                                                      │
│  kubectl scale rs nginx-rs --replicas=5                                             │
│                                                                                      │
│  After:  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                                              │
│          │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │  ← New pods created                          │
│          └───┘ └───┘ └───┘ └───┘ └───┘                                              │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Scale Down: replicas 5 → 2                                                         │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Before: ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                                              │
│          │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │                                              │
│          └───┘ └───┘ └───┘ └───┘ └───┘                                              │
│                                                                                      │
│  kubectl scale rs nginx-rs --replicas=2                                             │
│                                                                                      │
│  After:  ┌───┐ ┌───┐                                                                │
│          │ 1 │ │ 2 │  ← 3 pods terminated (newest first by default)                │
│          └───┘ └───┘                                                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Scale to Zero

```bash
# Scale to zero (stop all pods but keep RS)
kubectl scale rs nginx-rs --replicas=0

# Useful for:
# - Maintenance
# - Cost savings (dev environments)
# - Quick disable without deleting
```

---

## 📋 Pod Template

### What is the Pod Template?

> The **Pod Template** is the blueprint that ReplicaSet uses to create new pods. It's embedded in the ReplicaSet spec and defines exactly how each pod should be configured.

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  
  # This is the Pod Template
  template:
    metadata:
      labels:
        app: nginx
        version: v1
      annotations:
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
        env:
        - name: ENV
          value: production
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

### ⚠️ Important: Template Changes Don't Affect Existing Pods

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         TEMPLATE CHANGES                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  If you update the pod template in a ReplicaSet:                                   │
│                                                                                      │
│  1. Existing pods are NOT affected                                                 │
│  2. Only NEW pods will use the new template                                        │
│  3. You must delete old pods manually for them to be recreated                     │
│                                                                                      │
│  Example:                                                                           │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Before: image: nginx:1.18                                                          │
│          ┌───────────┐ ┌───────────┐ ┌───────────┐                                  │
│          │nginx:1.18 │ │nginx:1.18 │ │nginx:1.18 │                                  │
│          └───────────┘ └───────────┘ └───────────┘                                  │
│                                                                                      │
│  Update template: image: nginx:1.19                                                │
│          ┌───────────┐ ┌───────────┐ ┌───────────┐                                  │
│          │nginx:1.18 │ │nginx:1.18 │ │nginx:1.18 │  ← Still old version!           │
│          └───────────┘ └───────────┘ └───────────┘                                  │
│                                                                                      │
│  Delete one pod, RS creates new:                                                   │
│          ┌───────────┐ ┌───────────┐ ┌───────────┐                                  │
│          │nginx:1.18 │ │nginx:1.18 │ │nginx:1.19 │  ← Only new pod updated         │
│          └───────────┘ └───────────┘ └───────────┘                                  │
│                                                                                      │
│  This is why Deployments are preferred - they handle this automatically!           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ How ReplicaSets Work

### Pod Acquisition

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         POD ACQUISITION                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ReplicaSet can "adopt" existing pods if they match the selector!                  │
│                                                                                      │
│  Scenario: Orphan pods exist before ReplicaSet is created                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Step 1: Orphan pods exist                                                          │
│  ┌──────────────────┐  ┌──────────────────┐                                        │
│  │ Pod (no owner)   │  │ Pod (no owner)   │                                        │
│  │ labels:          │  │ labels:          │                                        │
│  │   app: nginx     │  │   app: nginx     │                                        │
│  └──────────────────┘  └──────────────────┘                                        │
│                                                                                      │
│  Step 2: Create ReplicaSet with replicas=3, selector: app=nginx                    │
│                                                                                      │
│  Step 3: ReplicaSet adopts existing pods + creates 1 more                          │
│  ┌──────────────────────────────────────────────────────────────┐                  │
│  │ ReplicaSet: nginx-rs (replicas: 3)                           │                  │
│  │                                                              │                  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │                  │
│  │  │ Pod (adopted)│  │ Pod (adopted)│  │ Pod (created)│       │                  │
│  │  │ app: nginx   │  │ app: nginx   │  │ app: nginx   │       │                  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │                  │
│  │                                                              │                  │
│  └──────────────────────────────────────────────────────────────┘                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Pod Termination Order

When scaling down, ReplicaSet uses this priority to decide which pods to delete:

| Priority | Condition | Reason |
|----------|-----------|--------|
| 1 | Pending pods | Not running yet |
| 2 | Pods on overloaded nodes | Balance load |
| 3 | Newer pods | Preserve stability |
| 4 | Random selection | If all else equal |

---

## 🔄 ReplicaSet vs Deployment

### Key Differences

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         REPLICASET VS DEPLOYMENT                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Feature              │ ReplicaSet               │ Deployment                       │
│  ─────────────────────┼──────────────────────────┼──────────────────────────────────│
│  Purpose              │ Maintain pod replicas    │ Manage ReplicaSets               │
│  Rolling Updates      │ ❌ No                    │ ✅ Yes                           │
│  Rollback             │ ❌ No                    │ ✅ Yes (revision history)        │
│  Update Strategy      │ Manual (delete pods)     │ Automatic (RollingUpdate)        │
│  Version History      │ ❌ No                    │ ✅ Yes                           │
│  Pause/Resume         │ ❌ No                    │ ✅ Yes                           │
│  Declarative Updates  │ Limited                  │ Full support                     │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Deployment manages ReplicaSet(s):                                                  │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────┐            │
│  │ Deployment                                                          │            │
│  │                                                                     │            │
│  │   ┌──────────────────────┐    ┌──────────────────────┐             │            │
│  │   │ ReplicaSet (old)     │    │ ReplicaSet (new)     │             │            │
│  │   │ replicas: 0          │    │ replicas: 3          │             │            │
│  │   │ (kept for rollback)  │    │ (current)            │             │            │
│  │   └──────────────────────┘    └──────────────────────┘             │            │
│  │                                                                     │            │
│  └────────────────────────────────────────────────────────────────────┘            │
│                                                                                      │
│  RECOMMENDATION: Almost always use Deployment instead of ReplicaSet directly!      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### When to Use ReplicaSet Directly?

| Use Case | Recommended |
|----------|-------------|
| Production workloads | Use **Deployment** |
| Rolling updates needed | Use **Deployment** |
| Need rollback | Use **Deployment** |
| Custom update logic | Maybe **ReplicaSet** |
| Learning/Understanding | **ReplicaSet** (then move to Deployment) |
| CKA Exam | Know both! |

---

## 🏷️ Ownership and Labels

### OwnerReferences

```yaml
# When ReplicaSet creates a pod, it adds ownerReferences
apiVersion: v1
kind: Pod
metadata:
  name: nginx-rs-abc123
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: nginx-rs
    uid: 12345-abcde
    controller: true
    blockOwnerDeletion: true
```

```bash
# View pod owner
kubectl get pod nginx-rs-abc123 -o jsonpath='{.metadata.ownerReferences}'

# View all pods with their owners
kubectl get pods -o custom-columns='NAME:.metadata.name,OWNER:.metadata.ownerReferences[0].name'
```

### Cascading Delete

```bash
# Delete ReplicaSet and all its pods (default)
kubectl delete rs nginx-rs

# Delete ReplicaSet but keep pods (orphan them)
kubectl delete rs nginx-rs --cascade=orphan

# Orphaned pods can be adopted by another RS with matching selector
```

---

## 🔧 Common Operations

### Complete Operations Reference

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl apply -f replicaset.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get rs                          # List all
kubectl get rs nginx-rs                 # Specific
kubectl get rs -o wide                  # More details
kubectl get rs -o yaml                  # Full YAML
kubectl get rs --show-labels            # Show labels

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe rs nginx-rs

# ═══════════════════════════════════════════════════════════════════
# SCALE
# ═══════════════════════════════════════════════════════════════════
kubectl scale rs nginx-rs --replicas=5
kubectl scale rs nginx-rs --replicas=0    # Stop all pods

# ═══════════════════════════════════════════════════════════════════
# EDIT
# ═══════════════════════════════════════════════════════════════════
kubectl edit rs nginx-rs

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete rs nginx-rs                  # Delete RS and pods
kubectl delete rs nginx-rs --cascade=orphan # Keep pods

# ═══════════════════════════════════════════════════════════════════
# VIEW PODS
# ═══════════════════════════════════════════════════════════════════
kubectl get pods -l app=nginx               # By label
kubectl get pods -o wide                    # With node info
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Pods not created | Selector doesn't match template | Ensure labels match |
| Wrong number of pods | Existing pods with same labels | Check for orphan pods |
| Pods pending | Insufficient resources | Check node resources |
| Image pull errors | Wrong image or no access | Verify image name |

### Debug Commands

```bash
# Check ReplicaSet status
kubectl get rs nginx-rs -o wide

# View events
kubectl describe rs nginx-rs | grep -A 20 "Events"

# Check pods created by RS
kubectl get pods -l app=nginx -o wide

# View pod issues
kubectl describe pod nginx-rs-xxxxx

# Check replica count
kubectl get rs nginx-rs -o jsonpath='{.status.replicas}'
kubectl get rs nginx-rs -o jsonpath='{.status.readyReplicas}'
kubectl get rs nginx-rs -o jsonpath='{.status.availableReplicas}'
```

---

## 🎓 CKA Exam Tips

### Quick ReplicaSet Creation

```bash
# Generate from scratch
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
EOF

# Quick scale
kubectl scale rs nginx-rs --replicas=5
```

### Key Points for Exam

1. **Selector must match template labels** - this is checked at creation
2. **Template changes don't update existing pods** - unlike Deployments
3. **Know the difference** between ReplicaSet and Deployment
4. **Scaling** is commonly tested: `kubectl scale rs <name> --replicas=<n>`
5. **Use Deployments** in practice, but understand ReplicaSets

### Exam Scenarios

```bash
# Scenario 1: Create RS with 4 replicas
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-rs
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx
EOF

# Scenario 2: Scale existing RS
kubectl scale rs my-rs --replicas=6

# Scenario 3: Fix broken RS (selector mismatch)
# Edit and ensure selector.matchLabels == template.metadata.labels
kubectl edit rs my-rs
```

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **ReplicaSet** | Maintains specified number of pod replicas |
| **Selector** | Defines which pods RS manages |
| **Template** | Blueprint for creating new pods |
| **Scaling** | Change replicas to add/remove pods |
| **Control Loop** | Observe → Compare → Act |

### ReplicaSet Formula

```
ReplicaSet = Desired Replicas + Pod Template + Label Selector
```

### When to Use What

| Scenario | Use |
|----------|-----|
| Simple pod replication | ReplicaSet (but usually Deployment) |
| Rolling updates needed | Deployment |
| Rollback capability | Deployment |
| Production workloads | Deployment |
| CKA exam questions | Know both! |

---

## 🔜 What's Next

In **Chapter 08: Deployments**, we'll cover:

- Why Deployments are the standard
- Rolling updates and rollbacks
- Deployment strategies
- Revision history
- Pausing and resuming

---

*ReplicaSets are the foundation - Deployments build on them for production use!*

