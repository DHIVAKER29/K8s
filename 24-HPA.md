# Chapter 24: Horizontal Pod Autoscaler (HPA)

## Introduction

Static replica counts don't match dynamic workloads. Traffic spikes during peak hours, drops at night. **Horizontal Pod Autoscaler (HPA)** automatically adjusts the number of pod replicas based on observed metrics.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE SCALING PROBLEM                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Static Replicas (Problem):                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   replicas: 3 (always)                                          │   │
│   │                                                                 │   │
│   │   Normal Load:        Peak Load:          Night:                │   │
│   │   ┌─┐ ┌─┐ ┌─┐        ┌─┐ ┌─┐ ┌─┐        ┌─┐ ┌─┐ ┌─┐           │   │
│   │   │█│ │█│ │█│        │█│ │█│ │█│        │░│ │░│ │░│           │   │
│   │   │█│ │█│ │█│        │█│ │█│ │█│        │░│ │░│ │░│           │   │
│   │   └─┘ └─┘ └─┘        └─┘ └─┘ └─┘        └─┘ └─┘ └─┘           │   │
│   │   50% CPU each       100% CPU!          5% CPU                 │   │
│   │   ✓ OK               ❌ Overloaded      ❌ Wasting $$$         │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   HPA (Solution):                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   replicas: auto (2-10 based on CPU)                            │   │
│   │                                                                 │   │
│   │   Normal Load:        Peak Load:          Night:                │   │
│   │   ┌─┐ ┌─┐ ┌─┐        ┌─┐┌─┐┌─┐┌─┐┌─┐    ┌─┐ ┌─┐               │   │
│   │   │█│ │█│ │█│        │█││█││█││█││█│    │█│ │█│               │   │
│   │   │█│ │█│ │█│        │█││█││█││█││█│    │█│ │█│               │   │
│   │   └─┘ └─┘ └─┘        └─┘└─┘└─┘└─┘└─┘    └─┘ └─┘               │   │
│   │   3 pods, 50%        8 pods, 60%        2 pods, 50%            │   │
│   │   ✓ Optimal          ✓ Scaled up        ✓ Scaled down          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. How HPA Works

### 1.1 HPA Control Loop

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HPA CONTROL LOOP                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Every 15 seconds (default):                                           │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   1. FETCH METRICS                                              │   │
│   │      │                                                          │   │
│   │      │  HPA Controller ──▶ Metrics Server                       │   │
│   │      │                     "What's the CPU usage?"              │   │
│   │      │                                                          │   │
│   │      ▼                                                          │   │
│   │   2. CALCULATE DESIRED REPLICAS                                 │   │
│   │      │                                                          │   │
│   │      │  desiredReplicas = ceil(currentReplicas *                │   │
│   │      │                    (currentMetricValue / targetValue))   │   │
│   │      │                                                          │   │
│   │      │  Example: 3 pods at 80% CPU, target 50%                  │   │
│   │      │  desired = ceil(3 * (80/50)) = ceil(4.8) = 5            │   │
│   │      │                                                          │   │
│   │      ▼                                                          │   │
│   │   3. SCALE DEPLOYMENT                                           │   │
│   │      │                                                          │   │
│   │      │  HPA Controller ──▶ Deployment                           │   │
│   │      │                     "Scale to 5 replicas"                │   │
│   │      │                                                          │   │
│   │      ▼                                                          │   │
│   │   4. WAIT (cooldown period)                                     │   │
│   │      │                                                          │   │
│   │      │  Prevent thrashing with stabilization window             │   │
│   │      │                                                          │   │
│   │      └──▶ Loop back to step 1                                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Prerequisites

| Requirement | Description |
|-------------|-------------|
| **Metrics Server** | Must be installed in cluster |
| **Resource Requests** | Pods must have CPU/memory requests |
| **Scalable Target** | Deployment, ReplicaSet, or StatefulSet |

```bash
# Check if metrics server is installed
kubectl get deployment metrics-server -n kube-system

# If not installed (for testing)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 2. Creating HPA

### 2.1 Imperative Command

```bash
# Create HPA targeting 50% CPU
kubectl autoscale deployment web-app \
  --cpu-percent=50 \
  --min=2 \
  --max=10

# View HPA
kubectl get hpa

# Describe HPA
kubectl describe hpa web-app
```

### 2.2 Declarative YAML (v2 API)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
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

### 2.3 Target Deployment (Must Have Requests)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3                    # Initial replicas (HPA will adjust)
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        resources:
          requests:              # REQUIRED for HPA!
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

---

## 3. Metric Types

### 3.1 Resource Metrics (CPU/Memory)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50    # Target 50% of requested CPU
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70    # Target 70% of requested memory
```

### 3.2 Target Types

| Target Type | Description | Example |
|-------------|-------------|---------|
| `Utilization` | Percentage of resource request | 50% of 100m CPU |
| `AverageValue` | Absolute value per pod | 100m CPU per pod |
| `Value` | Total value across all pods | 1000m total CPU |

```yaml
# Utilization - percentage of request
target:
  type: Utilization
  averageUtilization: 50

# AverageValue - absolute per pod
target:
  type: AverageValue
  averageValue: 100m

# Value - total across pods (for external metrics)
target:
  type: Value
  value: 1000
```

### 3.3 Pods Metrics (Custom per-pod metrics)

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: packets-per-second
    target:
      type: AverageValue
      averageValue: 1000
```

### 3.4 Object Metrics (Kubernetes objects)

```yaml
metrics:
- type: Object
  object:
    metric:
      name: requests-per-second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: web-ingress
    target:
      type: Value
      value: 10000
```

### 3.5 External Metrics

```yaml
metrics:
- type: External
  external:
    metric:
      name: queue_messages_ready
      selector:
        matchLabels:
          queue: my-queue
    target:
      type: AverageValue
      averageValue: 30
```

---

## 4. Scaling Behavior

### 4.1 Behavior Configuration (v2)

Control how fast HPA scales up and down:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: controlled-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately
      policies:
      - type: Percent
        value: 100                       # Double pods
        periodSeconds: 15
      - type: Pods
        value: 4                         # Or add 4 pods
        periodSeconds: 15
      selectPolicy: Max                  # Use whichever adds more
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 10                        # Remove 10% of pods
        periodSeconds: 60
      selectPolicy: Min                  # Use whichever removes fewer
```

### 4.2 Behavior Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `stabilizationWindowSeconds` | Time to wait before scaling | 300s (down), 0s (up) |
| `selectPolicy` | `Max`, `Min`, or `Disabled` | Max |
| `policies[].type` | `Pods` or `Percent` | - |
| `policies[].value` | Number of pods or percentage | - |
| `policies[].periodSeconds` | Time period for the policy | - |

### 4.3 Common Patterns

```yaml
# Aggressive scale up, slow scale down
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
  scaleDown:
    stabilizationWindowSeconds: 600    # 10 minutes
    policies:
    - type: Pods
      value: 1
      periodSeconds: 300               # Remove 1 pod per 5 min

# Conservative scaling (avoid thrashing)
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60
    policies:
    - type: Pods
      value: 2
      periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Pods
      value: 1
      periodSeconds: 120
```

---

## 5. Multiple Metrics

### 5.1 Scaling on Multiple Metrics

When using multiple metrics, HPA calculates desired replicas for each and uses the **highest** value:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 15
  metrics:
  # Scale if CPU > 50%
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  # OR if memory > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  # OR if requests > 1000/s per pod
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
```

---

## 6. HPA Status and Monitoring

### 6.1 View HPA Status

```bash
# List HPAs
kubectl get hpa

# Output:
# NAME          REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# web-app-hpa   Deployment/web   45%/50%   2         10        3          1h

# Detailed view
kubectl describe hpa web-app-hpa

# YAML output with status
kubectl get hpa web-app-hpa -o yaml
```

### 6.2 Understanding HPA Output

```
NAME          REFERENCE           TARGETS           MINPODS   MAXPODS   REPLICAS
web-app-hpa   Deployment/web-app  cpu: 45%/50%      2         10        3
                                  ^^^^     ^^^^
                                  current  target
```

### 6.3 HPA Events

```bash
# View HPA events
kubectl describe hpa web-app-hpa

# Events section shows:
# - ScaledUp: Scaled to X replicas
# - ScaledDown: Scaled to Y replicas
# - FailedGetResourceMetric: Can't get metrics
```

---

## 7. Complete Example

### 7.1 Full HPA Setup

```yaml
# Deployment with resource requests
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

### 7.2 Generate Load to Test

```bash
# Watch HPA in one terminal
kubectl get hpa php-apache-hpa -w

# Generate load in another terminal
kubectl run -i --tty load-generator --rm --image=busybox \
  --restart=Never -- /bin/sh -c \
  "while sleep 0.01; do wget -q -O- http://php-apache; done"

# Watch pods scale up
kubectl get pods -l app=php-apache -w

# Stop load generator (Ctrl+C) and watch scale down
```

---

## 8. Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HPA BEST PRACTICES                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Always set resource requests on containers                   │   │
│   │  • Set target below 100% (50-70% is typical)                   │   │
│   │  • Use stabilization window to prevent thrashing               │   │
│   │  • Start conservative, tune based on observation               │   │
│   │  • Monitor HPA events for issues                               │   │
│   │  • Test scaling behavior before production                     │   │
│   │  • Consider using Pod Disruption Budgets with HPA              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Use HPA without resource requests (won't work)              │   │
│   │  • Set min = max (defeats the purpose)                         │   │
│   │  • Target 100% utilization (no headroom)                       │   │
│   │  • Ignore scale down behavior (wastes resources)               │   │
│   │  • Use HPA on pods with persistent local state                 │   │
│   │  • Forget about Cluster Autoscaler interaction                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Command Reference

```bash
# Create HPA imperatively
kubectl autoscale deployment web --cpu-percent=50 --min=2 --max=10

# List HPAs
kubectl get hpa
kubectl get hpa -A                    # All namespaces

# Describe HPA
kubectl describe hpa <name>

# Get HPA YAML
kubectl get hpa <name> -o yaml

# Delete HPA
kubectl delete hpa <name>

# Watch HPA changes
kubectl get hpa -w

# Check metrics server
kubectl top pods
kubectl top nodes
```

---

## 10. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Create HPA | 🔴 HIGH | Imperative command |
| CPU-based scaling | 🔴 HIGH | Basic HPA YAML |
| View HPA status | 🟡 MEDIUM | kubectl get/describe |
| Scaling behavior | 🟡 MEDIUM | v2 API behavior |

### Quick Reference for Exam

```bash
# Quick HPA creation
kubectl autoscale deployment web --cpu-percent=50 --min=2 --max=10
```

```yaml
# Minimal HPA YAML
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
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

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HPA SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   FORMULA:                                                              │
│   desiredReplicas = ceil(currentReplicas × (currentValue / targetValue))│
│                                                                         │
│   REQUIREMENTS:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Metrics Server installed                                     │   │
│   │  • Resource requests on containers                              │   │
│   │  • Scalable target (Deployment, StatefulSet)                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   METRIC TYPES:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Resource: CPU, memory (built-in)                            │   │
│   │  • Pods: Custom per-pod metrics                                │   │
│   │  • Object: Kubernetes object metrics                           │   │
│   │  • External: External system metrics                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Custom Resources (CRDs)** - Extending Kubernetes
- Creating Custom Resource Definitions
- Custom controllers and operators

---

**Chapter 24 Complete! 🎉**

You now understand:
- How HPA works (control loop)
- Creating HPA (imperative and declarative)
- Metric types (Resource, Pods, Object, External)
- Scaling behavior configuration
- Multiple metrics scaling
- Best practices
- CKA exam preparation

