# ☸️ Chapter 08: Deployments

> The standard way to run applications in Kubernetes - declarative updates, rolling deployments, and instant rollbacks.

---

## 📚 Table of Contents

1. [What is a Deployment?](#-what-is-a-deployment)
2. [Why Deployments?](#-why-deployments)
3. [Deployment Architecture](#-deployment-architecture)
4. [Creating Deployments](#-creating-deployments)
5. [Deployment Manifest](#-deployment-manifest)
6. [Rolling Updates](#-rolling-updates)
7. [Rollback](#-rollback)
8. [Deployment Strategies](#-deployment-strategies)
9. [Scaling Deployments](#-scaling-deployments)
10. [Pausing and Resuming](#-pausing-and-resuming)
11. [Deployment Status](#-deployment-status)
12. [Advanced Configuration](#-advanced-configuration)
13. [Common Operations](#-common-operations)
14. [Troubleshooting](#-troubleshooting)
15. [CKA Exam Tips](#-cka-exam-tips)
16. [Summary](#-summary)

---

## 📖 What is a Deployment?

### Definition

> **Deployment** is a Kubernetes resource that provides declarative updates for Pods and ReplicaSets. It manages the rollout of new versions and enables easy rollback to previous versions.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS A DEPLOYMENT?                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Deployment = ReplicaSet Manager + Update Strategy + Rollback Capability            │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │                            Deployment                                          ││
│  │                                                                                ││
│  │  • Manages ReplicaSets                                                        ││
│  │  • Provides rolling updates                                                   ││
│  │  • Enables instant rollback                                                   ││
│  │  • Maintains revision history                                                 ││
│  │                                                                                ││
│  │  ┌────────────────────────────┐    ┌────────────────────────────┐             ││
│  │  │   ReplicaSet (revision 1)  │    │   ReplicaSet (revision 2)  │             ││
│  │  │   replicas: 0 (old)        │    │   replicas: 3 (current)    │             ││
│  │  │   image: nginx:1.18        │    │   image: nginx:1.19        │             ││
│  │  │                            │    │                            │             ││
│  │  │   (kept for rollback)      │    │   ┌───┐ ┌───┐ ┌───┐       │             ││
│  │  │                            │    │   │Pod│ │Pod│ │Pod│       │             ││
│  │  │                            │    │   └───┘ └───┘ └───┘       │             ││
│  │  └────────────────────────────┘    └────────────────────────────┘             ││
│  │                                                                                ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Deployment Hierarchy

```
Deployment
    │
    ├── Creates and manages ──► ReplicaSet (revision 1)
    │                               │
    │                               └── Manages ──► Pods
    │
    └── On update, creates ──► ReplicaSet (revision 2)
                                    │
                                    └── Manages ──► Pods (new version)
```

---

## ❓ Why Deployments?

### Problems Solved by Deployments

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         WHY DEPLOYMENTS?                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  PROBLEM 1: Manual Updates with ReplicaSets                                         │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  With ReplicaSet only:                                                              │
│  1. Update template                      ← Change image to nginx:1.19              │
│  2. Existing pods still run old version  ← nginx:1.18 still running!              │
│  3. Must manually delete pods            ← Tedious and error-prone                 │
│  4. No automatic rollback                ← If new version breaks, manual fix       │
│                                                                                      │
│  With Deployment:                                                                   │
│  1. Update Deployment spec               ← Change image to nginx:1.19              │
│  2. Automatic rolling update             ← Old pods replaced gradually             │
│  3. Zero downtime                        ← Always some pods available              │
│  4. Easy rollback                        ← kubectl rollout undo                    │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  PROBLEM 2: Downtime During Updates                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Without rolling update:                                                            │
│  ┌───┐ ┌───┐ ┌───┐          (all down)         ┌───┐ ┌───┐ ┌───┐                   │
│  │v1 │ │v1 │ │v1 │  ────►   ❌ DOWNTIME  ────► │v2 │ │v2 │ │v2 │                   │
│  └───┘ └───┘ └───┘                              └───┘ └───┘ └───┘                   │
│                                                                                      │
│  With Deployment rolling update:                                                    │
│  ┌───┐ ┌───┐ ┌───┐    ┌───┐ ┌───┐ ┌───┐    ┌───┐ ┌───┐ ┌───┐                       │
│  │v1 │ │v1 │ │v1 │ ─► │v1 │ │v1 │ │v2 │ ─► │v1 │ │v2 │ │v2 │ ─► │v2 │ │v2 │ │v2 │ │
│  └───┘ └───┘ └───┘    └───┘ └───┘ └───┘    └───┘ └───┘ └───┘                       │
│                         ↑ Always some pods running!                                 │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  PROBLEM 3: Bad Deployment Recovery                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Deployed v2 but it's broken?                                                       │
│  With Deployment: kubectl rollout undo deployment/nginx                            │
│  Instantly rolls back to v1!                                                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Deployment Capabilities

| Capability | Description |
|------------|-------------|
| **Rolling Updates** | Gradual replacement of pods |
| **Rollback** | Instant revert to previous version |
| **Scaling** | Easy horizontal scaling |
| **Pause/Resume** | Pause rollout, make changes, resume |
| **Revision History** | Track all deployment versions |
| **Self-Healing** | Automatically replace failed pods |

---

## 🏗️ Deployment Architecture

### How Deployments Work

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DEPLOYMENT ARCHITECTURE                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. You create/update Deployment                                                    │
│                                                                                      │
│     kubectl apply -f deployment.yaml                                                │
│              │                                                                       │
│              ▼                                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                          API Server                                           │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│              │                                                                       │
│              ▼                                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                    Deployment Controller                                      │ │
│  │                                                                               │ │
│  │   "I need to ensure desired state matches actual state"                      │ │
│  │                                                                               │ │
│  │   • Creates/updates ReplicaSets                                              │ │
│  │   • Manages rolling updates                                                  │ │
│  │   • Handles rollbacks                                                        │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│              │                                                                       │
│              ▼                                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                         ReplicaSet                                            │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│              │                                                                       │
│              ▼                                                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                     │
│  │      Pod        │  │      Pod        │  │      Pod        │                     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Creating Deployments

### Imperative Creation

```bash
# Create deployment
kubectl create deployment nginx --image=nginx

# Create with replicas
kubectl create deployment nginx --image=nginx --replicas=3

# Create with port
kubectl create deployment nginx --image=nginx --port=80

# Generate YAML (dry-run)
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deployment.yaml

# Create and expose in one go
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=ClusterIP
```

### Declarative Creation

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
```

---

## 📝 Deployment Manifest

### Complete Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
    tier: frontend
  annotations:
    kubernetes.io/change-cause: "Initial deployment"
spec:
  # Number of pod replicas
  replicas: 3
  
  # How to find pods to manage
  selector:
    matchLabels:
      app: nginx
  
  # Update strategy
  strategy:
    type: RollingUpdate           # or Recreate
    rollingUpdate:
      maxSurge: 1                 # Max extra pods during update
      maxUnavailable: 0           # Max pods that can be unavailable
  
  # Revision history limit
  revisionHistoryLimit: 10        # Keep 10 old ReplicaSets
  
  # Minimum ready seconds
  minReadySeconds: 5              # Wait 5s before considering pod ready
  
  # Progress deadline
  progressDeadlineSeconds: 600    # 10 minutes to complete rollout
  
  # Pod template
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
          name: http
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
        env:
        - name: ENV
          value: "production"
      terminationGracePeriodSeconds: 30
```

### Key Fields Explained

| Field | Description | Default |
|-------|-------------|---------|
| `replicas` | Number of pod copies | 1 |
| `selector` | Labels to match pods | Required |
| `strategy.type` | RollingUpdate or Recreate | RollingUpdate |
| `maxSurge` | Extra pods during update | 25% |
| `maxUnavailable` | Pods that can be down | 25% |
| `revisionHistoryLimit` | Old ReplicaSets to keep | 10 |
| `minReadySeconds` | Wait time before ready | 0 |
| `progressDeadlineSeconds` | Rollout timeout | 600 |

---

## 🔄 Rolling Updates

### How Rolling Updates Work

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ROLLING UPDATE PROCESS                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Configuration: replicas=3, maxSurge=1, maxUnavailable=0                            │
│  Update: image nginx:1.18 → nginx:1.19                                              │
│                                                                                      │
│  Step 1: Initial State                                                              │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Old RS (nginx:1.18): ┌───┐ ┌───┐ ┌───┐                                             │
│                       │v18│ │v18│ │v18│   replicas: 3                               │
│                       └───┘ └───┘ └───┘                                             │
│  New RS (nginx:1.19):                     replicas: 0                               │
│                                                                                      │
│  Step 2: Create new pod (maxSurge=1 allows 4 total)                                │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Old RS: ┌───┐ ┌───┐ ┌───┐                replicas: 3                               │
│          │v18│ │v18│ │v18│                                                          │
│          └───┘ └───┘ └───┘                                                          │
│  New RS:             ┌───┐                replicas: 1                               │
│                      │v19│ ← New pod starting                                       │
│                      └───┘                                                          │
│                                                                                      │
│  Step 3: New pod ready, terminate old pod                                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Old RS: ┌───┐ ┌───┐                      replicas: 2                               │
│          │v18│ │v18│                                                                │
│          └───┘ └───┘                                                                │
│  New RS: ┌───┐ ┌───┐                      replicas: 2                               │
│          │v19│ │v19│                                                                │
│          └───┘ └───┘                                                                │
│                                                                                      │
│  Step 4: Continue until complete                                                    │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Old RS:                                  replicas: 0 (kept for rollback)           │
│  New RS: ┌───┐ ┌───┐ ┌───┐               replicas: 3                               │
│          │v19│ │v19│ │v19│                                                          │
│          └───┘ └───┘ └───┘                                                          │
│                                                                                      │
│  ✅ Rollout complete! Zero downtime achieved.                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Triggering Updates

```bash
# Method 1: Update image
kubectl set image deployment/nginx nginx=nginx:1.20

# Method 2: Edit deployment
kubectl edit deployment nginx

# Method 3: Apply updated YAML
kubectl apply -f deployment.yaml

# Method 4: Patch
kubectl patch deployment nginx -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.20"}]}}}}'

# Add change cause annotation (for history)
kubectl annotate deployment nginx kubernetes.io/change-cause="Update to nginx 1.20"
```

### Monitor Rollout

```bash
# Watch rollout status
kubectl rollout status deployment/nginx

# Watch rollout with progress
kubectl rollout status deployment/nginx -w

# View rollout history
kubectl rollout history deployment/nginx

# View specific revision
kubectl rollout history deployment/nginx --revision=2
```

---

## ⏪ Rollback

### Rolling Back Deployments

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ROLLBACK PROCESS                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Deployment keeps old ReplicaSets for rollback:                                     │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │                         Deployment: nginx                                      ││
│  │                                                                                ││
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                  ││
│  │  │ RS revision 1   │ │ RS revision 2   │ │ RS revision 3   │ ◄── Current     ││
│  │  │ nginx:1.18      │ │ nginx:1.19      │ │ nginx:1.20      │                  ││
│  │  │ replicas: 0     │ │ replicas: 0     │ │ replicas: 3     │                  ││
│  │  └─────────────────┘ └─────────────────┘ └─────────────────┘                  ││
│  │                                                                                ││
│  │  kubectl rollout undo deployment/nginx                                        ││
│  │                                                                                ││
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                  ││
│  │  │ RS revision 1   │ │ RS revision 4   │ │ RS revision 3   │                  ││
│  │  │ nginx:1.18      │ │ nginx:1.19      │ │ nginx:1.20      │                  ││
│  │  │ replicas: 0     │ │ replicas: 3     │ │ replicas: 0     │                  ││
│  │  └─────────────────┘ └─────────────────┘ └─────────────────┘                  ││
│  │                       ↑ Now current (was rev 2, now rev 4)                    ││
│  │                                                                                ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Rollback Commands

```bash
# Undo to previous revision
kubectl rollout undo deployment/nginx

# Undo to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# View history first
kubectl rollout history deployment/nginx
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Update to nginx 1.19
# 3         Update to nginx 1.20

# Rollback to revision 1
kubectl rollout undo deployment/nginx --to-revision=1
```

---

## 📊 Deployment Strategies

### RollingUpdate (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%          # 25% extra pods allowed
      maxUnavailable: 25%    # 25% pods can be down
```

### Strategy Configurations

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ROLLING UPDATE STRATEGIES                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. ZERO DOWNTIME (Conservative)                                                    │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  maxSurge: 1                                                                        │
│  maxUnavailable: 0                                                                  │
│                                                                                      │
│  • Always maintain full capacity                                                    │
│  • Slower rollout (one pod at a time)                                              │
│  • Requires extra resources during update                                          │
│                                                                                      │
│  replicas=3:  [v1][v1][v1] → [v1][v1][v1][v2] → [v1][v1][v2][v2] → [v1][v2][v2][v2]│
│                                                  → [v2][v2][v2]                     │
│                                                                                      │
│  2. FAST ROLLOUT (Aggressive)                                                       │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  maxSurge: 50%                                                                      │
│  maxUnavailable: 50%                                                                │
│                                                                                      │
│  • Fast update                                                                      │
│  • Temporary capacity reduction                                                     │
│  • Good for dev/staging                                                             │
│                                                                                      │
│  3. BALANCED (Default)                                                              │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  maxSurge: 25%                                                                      │
│  maxUnavailable: 25%                                                                │
│                                                                                      │
│  • Balance between speed and availability                                          │
│  • Good for most cases                                                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Recreate Strategy

```yaml
spec:
  strategy:
    type: Recreate
```

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         RECREATE STRATEGY                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  All old pods terminated before new pods created                                    │
│                                                                                      │
│  Before:    ┌───┐ ┌───┐ ┌───┐                                                       │
│             │v1 │ │v1 │ │v1 │                                                       │
│             └───┘ └───┘ └───┘                                                       │
│                                                                                      │
│  During:              (empty)        ← DOWNTIME!                                    │
│                                                                                      │
│  After:     ┌───┐ ┌───┐ ┌───┐                                                       │
│             │v2 │ │v2 │ │v2 │                                                       │
│             └───┘ └───┘ └───┘                                                       │
│                                                                                      │
│  Use when:                                                                          │
│  • Application doesn't support running multiple versions                           │
│  • Shared volume requires exclusive access                                         │
│  • Dev environments where downtime is acceptable                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📈 Scaling Deployments

### Scaling Commands

```bash
# Scale to specific replicas
kubectl scale deployment nginx --replicas=5

# Scale multiple deployments
kubectl scale deployment nginx redis --replicas=3

# Scale based on current (conditional)
kubectl scale deployment nginx --replicas=5 --current-replicas=3

# Scale to zero (pause workload)
kubectl scale deployment nginx --replicas=0
```

### Horizontal Pod Autoscaler (HPA)

```bash
# Create HPA
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# View HPA
kubectl get hpa

# Describe HPA
kubectl describe hpa nginx

# Delete HPA
kubectl delete hpa nginx
```

```yaml
# HPA manifest
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

---

## ⏸️ Pausing and Resuming

### Use Cases

- Make multiple changes without triggering multiple rollouts
- Fix issues during a rollout
- Wait for manual approval

### Commands

```bash
# Pause rollout
kubectl rollout pause deployment/nginx

# Make changes while paused
kubectl set image deployment/nginx nginx=nginx:1.20
kubectl set resources deployment/nginx -c=nginx --limits=memory=256Mi

# Resume rollout (applies all changes at once)
kubectl rollout resume deployment/nginx

# Check if paused
kubectl get deployment nginx -o jsonpath='{.spec.paused}'
```

---

## 📊 Deployment Status

### Status Fields

```bash
# Get deployment status
kubectl get deployment nginx -o wide

# Detailed status
kubectl describe deployment nginx

# JSON status
kubectl get deployment nginx -o jsonpath='{.status}'
```

### Status Conditions

| Condition | Meaning |
|-----------|---------|
| `Available` | Minimum pods available |
| `Progressing` | Rollout in progress or completed |
| `ReplicaFailure` | Failed to create/delete pods |

### Rollout Status

```bash
# Check rollout status
kubectl rollout status deployment/nginx

# Output examples:
# "deployment "nginx" successfully rolled out"
# "Waiting for deployment "nginx" rollout to finish: 1 of 3 updated replicas are available..."
# "error: deployment "nginx" exceeded its progress deadline"
```

---

## ⚙️ Advanced Configuration

### Min Ready Seconds

```yaml
spec:
  minReadySeconds: 10  # Pod must be ready for 10s before considered available
```

### Progress Deadline

```yaml
spec:
  progressDeadlineSeconds: 600  # Rollout must complete in 10 minutes
```

### Revision History Limit

```yaml
spec:
  revisionHistoryLimit: 5  # Keep only 5 old ReplicaSets
```

### Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2           # At least 2 pods must be available
  # OR
  # maxUnavailable: 1       # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: nginx
```

---

## 🔧 Common Operations

### Complete Operations Reference

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl create deployment nginx --image=nginx --replicas=3
kubectl apply -f deployment.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get deployments
kubectl get deploy                        # Short form
kubectl get deploy -o wide
kubectl get deploy nginx -o yaml

# ═══════════════════════════════════════════════════════════════════
# UPDATE
# ═══════════════════════════════════════════════════════════════════
kubectl set image deploy/nginx nginx=nginx:1.20
kubectl edit deploy nginx
kubectl apply -f deployment.yaml
kubectl patch deploy nginx -p '{"spec":{"replicas":5}}'

# ═══════════════════════════════════════════════════════════════════
# SCALE
# ═══════════════════════════════════════════════════════════════════
kubectl scale deploy nginx --replicas=5

# ═══════════════════════════════════════════════════════════════════
# ROLLOUT
# ═══════════════════════════════════════════════════════════════════
kubectl rollout status deploy/nginx
kubectl rollout history deploy/nginx
kubectl rollout undo deploy/nginx
kubectl rollout undo deploy/nginx --to-revision=2
kubectl rollout pause deploy/nginx
kubectl rollout resume deploy/nginx
kubectl rollout restart deploy/nginx     # Trigger new rollout

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete deploy nginx
kubectl delete -f deployment.yaml
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Rollout stuck | Pods failing readiness | Check pod logs, fix app |
| Progress deadline exceeded | Rollout too slow | Increase deadline or fix pods |
| ImagePullBackOff | Wrong image | Check image name/tag |
| CrashLoopBackOff | App crashing | Check logs, fix app |
| Insufficient resources | No capacity | Free resources or add nodes |

### Debug Commands

```bash
# Check deployment status
kubectl describe deploy nginx

# View events
kubectl get events --field-selector involvedObject.name=nginx

# Check ReplicaSets
kubectl get rs -l app=nginx

# Check pods
kubectl get pods -l app=nginx

# Pod logs
kubectl logs -l app=nginx

# Rollout status
kubectl rollout status deploy/nginx

# Recent changes
kubectl rollout history deploy/nginx
```

### Stuck Rollout Fix

```bash
# Option 1: Rollback
kubectl rollout undo deploy/nginx

# Option 2: Fix and continue
kubectl edit deploy nginx  # Fix the issue
kubectl rollout resume deploy/nginx

# Option 3: Force restart
kubectl rollout restart deploy/nginx
```

---

## 🎓 CKA Exam Tips

### Quick Deployment Creation

```bash
# Create deployment quickly
kubectl create deployment nginx --image=nginx --replicas=3

# Generate YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml

# Expose immediately
kubectl expose deployment nginx --port=80 --type=NodePort
```

### Common Exam Tasks

```bash
# Task: Create deployment with 3 replicas
kubectl create deployment webapp --image=nginx --replicas=3

# Task: Update image to specific version
kubectl set image deploy/webapp nginx=nginx:1.19

# Task: Rollback deployment
kubectl rollout undo deploy/webapp

# Task: Scale deployment
kubectl scale deploy/webapp --replicas=5

# Task: Check rollout status
kubectl rollout status deploy/webapp

# Task: View rollout history
kubectl rollout history deploy/webapp
```

### Key Commands to Memorize

```bash
# Update image
kubectl set image deploy/<name> <container>=<image>

# Rollback
kubectl rollout undo deploy/<name>
kubectl rollout undo deploy/<name> --to-revision=<n>

# Status
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>

# Scale
kubectl scale deploy/<name> --replicas=<n>
```

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Deployment** | Manages ReplicaSets with rolling updates |
| **Rolling Update** | Gradual pod replacement |
| **Rollback** | Revert to previous version |
| **Strategy** | RollingUpdate or Recreate |
| **maxSurge** | Extra pods during update |
| **maxUnavailable** | Pods that can be down |
| **Revision History** | Old ReplicaSets for rollback |

### Deployment vs ReplicaSet

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| Pod replication | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback | ❌ | ✅ |
| Revision history | ❌ | ✅ |
| Pause/Resume | ❌ | ✅ |
| **Use in production** | Rarely | Always |

### Essential Commands

```bash
kubectl create deployment <name> --image=<image> --replicas=<n>
kubectl set image deploy/<name> <container>=<image>
kubectl scale deploy/<name> --replicas=<n>
kubectl rollout status/history/undo deploy/<name>
```

---

## 🔜 What's Next

In **Chapter 09: DaemonSets**, we'll cover:

- Running a pod on every node
- Use cases (logging, monitoring, networking)
- Node selectors and tolerations
- Update strategies for DaemonSets

---

*Deployments are THE way to run applications in Kubernetes - master them!*

