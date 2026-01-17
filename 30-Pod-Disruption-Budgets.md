# Chapter 30: Pod Disruption Budgets (PDB)

## Introduction

When you drain a node for maintenance, Kubernetes evicts all pods. But what if all replicas of a critical application get evicted at once? **Pod Disruption Budgets (PDB)** prevent this by ensuring a minimum number of pods remain available during voluntary disruptions.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE DISRUPTION PROBLEM                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without PDB:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Node 1          Node 2          Node 3                       │   │
│   │   ┌─────┐         ┌─────┐         ┌─────┐                      │   │
│   │   │App-1│         │App-2│         │App-3│                      │   │
│   │   └─────┘         └─────┘         └─────┘                      │   │
│   │                                                                 │   │
│   │   kubectl drain node-1 node-2 node-3 (simultaneously)         │   │
│   │                                                                 │   │
│   │   Result: ALL pods evicted! Service DOWN! ❌                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With PDB (minAvailable: 2):                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Node 1          Node 2          Node 3                       │   │
│   │   ┌─────┐         ┌─────┐         ┌─────┐                      │   │
│   │   │App-1│         │App-2│         │App-3│                      │   │
│   │   └─────┘         └─────┘         └─────┘                      │   │
│   │                                                                 │   │
│   │   kubectl drain node-1 → Evicts App-1 ✓                        │   │
│   │   kubectl drain node-2 → BLOCKED! Only 2 pods left            │   │
│   │   (Must wait for App-1 to reschedule before continuing)        │   │
│   │                                                                 │   │
│   │   Result: Always ≥2 pods running! Service UP! ✓                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Understanding Disruptions

### 1.1 Voluntary vs Involuntary Disruptions

| Type | Description | Examples | PDB Applies? |
|------|-------------|----------|--------------|
| **Voluntary** | Intentional actions | Node drain, rolling update, pod delete | ✅ YES |
| **Involuntary** | Unplanned events | Node crash, OOM kill, hardware failure | ❌ NO |

### 1.2 Voluntary Disruptions

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VOLUNTARY DISRUPTIONS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Actions that trigger voluntary disruptions:                           │
│                                                                         │
│   • kubectl drain <node>           (node maintenance)                   │
│   • kubectl delete pod <pod>       (manual deletion)                    │
│   • Deployment rolling updates     (strategy: RollingUpdate)           │
│   • Cluster autoscaler scale-down  (removing nodes)                    │
│   • kubectl rollout restart        (restart deployment)                │
│                                                                         │
│   PDB protects against ALL of these!                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Creating Pod Disruption Budgets

### 2.1 PDB with minAvailable

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2              # At least 2 pods must be available
  selector:
    matchLabels:
      app: web                 # Applies to pods with this label
```

### 2.2 PDB with maxUnavailable

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 1            # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: web
```

### 2.3 Using Percentages

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: "50%"          # At least 50% must be available
  selector:
    matchLabels:
      app: web
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  maxUnavailable: "25%"        # At most 25% can be unavailable
  selector:
    matchLabels:
      app: api
```

---

## 3. minAvailable vs maxUnavailable

### 3.1 Comparison

| Setting | Meaning | Example (5 replicas) |
|---------|---------|---------------------|
| `minAvailable: 3` | Must keep ≥3 running | Can evict up to 2 |
| `maxUnavailable: 2` | Can disrupt ≤2 at once | Must keep ≥3 running |
| `minAvailable: "60%"` | Must keep ≥60% running | Keep ≥3 (60% of 5) |
| `maxUnavailable: "40%"` | Can disrupt ≤40% | Can evict ≤2 (40% of 5) |

### 3.2 Which to Use?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHICH TO USE?                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Use minAvailable when:                                                │
│   • You know the minimum pods needed for the service                   │
│   • Example: "I need at least 2 pods for HA"                           │
│                                                                         │
│   Use maxUnavailable when:                                              │
│   • You want to allow some disruption regardless of replica count      │
│   • Example: "I can tolerate losing 1 pod at a time"                   │
│                                                                         │
│   Use percentages when:                                                 │
│   • Replica count varies (HPA, different environments)                 │
│   • Example: "Always keep 80% available"                               │
│                                                                         │
│   ⚠️  You can only specify ONE: minAvailable OR maxUnavailable         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Complete Example

### 4.1 Deployment + PDB

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 3              # Always keep at least 3 pods
  selector:
    matchLabels:
      app: web                 # Must match deployment's pod labels
```

### 4.2 Apply and Verify

```bash
# Apply resources
kubectl apply -f deployment-with-pdb.yaml

# Check PDB status
kubectl get pdb

# Output:
# NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# web-pdb   3               N/A               2                     5s

# Detailed view
kubectl describe pdb web-pdb
```

---

## 5. PDB in Action

### 5.1 How Drain Respects PDB

```bash
# Scenario: 5 pods on 3 nodes, PDB minAvailable: 3

# Before drain:
# Node-1: pod-1, pod-2
# Node-2: pod-3, pod-4
# Node-3: pod-5

# Drain node-1:
kubectl drain node-1 --ignore-daemonsets

# Kubernetes checks PDB:
# - Current available: 5
# - Min required: 3
# - Can evict: 2 (5 - 3 = 2)
# - Node-1 has 2 pods → Allowed!

# After draining node-1:
# Available: 3 pods (on node-2 and node-3)
# New pods scheduled elsewhere

# Drain node-2 immediately after:
kubectl drain node-2 --ignore-daemonsets

# Kubernetes checks PDB:
# - Current available: 3 (still waiting for rescheduling)
# - Min required: 3
# - Can evict: 0
# - BLOCKED! Must wait for pods to reschedule
```

### 5.2 PDB Blocking Example

```bash
# If PDB blocks the drain:
$ kubectl drain node-2 --ignore-daemonsets
error: Cannot evict pod as it would violate the pod's disruption budget.

# Use --timeout to wait
kubectl drain node-2 --ignore-daemonsets --timeout=300s

# Or force (DANGEROUS - ignores PDB!)
kubectl drain node-2 --ignore-daemonsets --force --delete-emptydir-data
```

---

## 6. Command Reference

```bash
# Create PDB imperatively
kubectl create pdb my-pdb --selector=app=web --min-available=2

# Or with maxUnavailable
kubectl create pdb my-pdb --selector=app=web --max-unavailable=1

# List PDBs
kubectl get pdb
kubectl get pdb -A

# Describe PDB
kubectl describe pdb <pdb-name>

# Delete PDB
kubectl delete pdb <pdb-name>

# Check PDB status
kubectl get pdb -o wide
```

---

## 7. Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PDB BEST PRACTICES                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   • Create PDB for all production workloads                            │
│   • Match PDB selector to pod labels exactly                           │
│   • Set minAvailable < replicas (leave room for disruption)           │
│   • Use percentages for HPA-managed workloads                          │
│   • Test PDB before production (try kubectl drain)                     │
│                                                                         │
│   ❌ DON'T:                                                             │
│   • Set minAvailable = replicas (blocks all drains!)                   │
│   • Use both minAvailable AND maxUnavailable                           │
│   • Forget PDB when using cluster autoscaler                           │
│   • Create PDB for single-replica deployments (use ≥2 replicas)       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. CKA Exam Tips

### Quick Reference

```bash
# Create PDB imperatively (fastest for exam)
kubectl create pdb my-pdb --selector=app=web --min-available=2

# Check allowed disruptions
kubectl get pdb
```

### Common Exam Scenarios

| Scenario | Solution |
|----------|----------|
| "Ensure at least 2 pods always running" | `minAvailable: 2` |
| "Allow only 1 pod disruption at a time" | `maxUnavailable: 1` |
| "Keep 80% of pods available" | `minAvailable: "80%"` |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       PDB SUMMARY                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   PURPOSE: Ensure minimum availability during voluntary disruptions    │
│                                                                         │
│   OPTIONS:                                                              │
│   • minAvailable: Minimum pods that must stay running                  │
│   • maxUnavailable: Maximum pods that can be disrupted                 │
│   • Can use integers or percentages                                    │
│                                                                         │
│   PROTECTS AGAINST:                                                     │
│   • kubectl drain                                                      │
│   • Rolling updates                                                    │
│   • Manual pod deletion                                                │
│   • Cluster autoscaler                                                 │
│                                                                         │
│   DOES NOT PROTECT AGAINST:                                            │
│   • Node crashes                                                       │
│   • OOM kills                                                          │
│   • Hardware failures                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**Chapter 30 Complete!** ✅

