# ☸️ Chapter 14: Network Policies

> Securing pod-to-pod communication with firewall rules at the Kubernetes level.

---

## 📚 Table of Contents

1. [What are Network Policies?](#-what-are-network-policies)
2. [Why Network Policies?](#-why-network-policies)
3. [Network Policy Components](#-network-policy-components)
4. [Default Behavior](#-default-behavior)
5. [Creating Network Policies](#-creating-network-policies)
6. [Ingress Rules](#-ingress-rules)
7. [Egress Rules](#-egress-rules)
8. [Selectors](#-selectors)
9. [Namespace Isolation](#-namespace-isolation)
10. [Default Deny Policies](#-default-deny-policies)
11. [Common Patterns](#-common-patterns)
12. [CNI Requirements](#-cni-requirements)
13. [Operations](#-operations)
14. [Troubleshooting](#-troubleshooting)
15. [CKA Exam Tips](#-cka-exam-tips)
16. [Summary](#-summary)

---

## 📖 What are Network Policies?

### Definition

> **Network Policy** is a Kubernetes resource that controls traffic flow at the IP address or port level (OSI layer 3 or 4). It defines how groups of pods are allowed to communicate with each other and other network endpoints.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         WHAT IS A NETWORK POLICY?                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Network Policy = Firewall Rules for Pods                                           │
│                                                                                      │
│  Without Network Policy:              With Network Policy:                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌─────┐     ┌─────┐                 ┌─────┐     ┌─────┐                            │
│  │Web  │◄───►│ DB  │                 │Web  │────►│ DB  │                            │
│  └─────┘     └─────┘                 └─────┘     └──┬──┘                            │
│     │           │                       │          ✓│                               │
│     │           │                       │           │                               │
│  ┌──┴──┐     ┌──┴──┐                 ┌──┴──┐     ┌──┴──┐                            │
│  │Cache│◄───►│Hack │                 │Cache│  ✗  │Hack │                            │
│  └─────┘     └─────┘                 └─────┘     └─────┘                            │
│                                                                                      │
│  Everything can talk               Only allowed traffic                             │
│  to everything!                    can reach DB                                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Network Policy Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Pod-level** | Rules apply to specific pods |
| **Namespace-scoped** | Policies are namespaced |
| **Additive** | Multiple policies combine (union) |
| **Stateful** | Return traffic automatically allowed |
| **Label-based** | Select pods by labels |

---

## ❓ Why Network Policies?

### Security Benefits

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         WHY NETWORK POLICIES?                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. DEFENSE IN DEPTH                                                                │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  Even if an attacker compromises one pod, they can't reach other pods              │
│                                                                                      │
│  Compromised Pod                                                                    │
│       │                                                                              │
│       ├──► DB Pod: ❌ BLOCKED by Network Policy                                    │
│       ├──► Secret Store: ❌ BLOCKED by Network Policy                              │
│       └──► Other Services: ❌ BLOCKED by Network Policy                            │
│                                                                                      │
│  2. COMPLIANCE                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • PCI-DSS: Segment cardholder data                                                │
│  • HIPAA: Isolate patient data                                                     │
│  • SOC2: Control data access                                                       │
│                                                                                      │
│  3. MULTI-TENANT ISOLATION                                                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Tenant A's pods can't access Tenant B's pods                                    │
│  • Different teams in different namespaces                                         │
│                                                                                      │
│  4. LEAST PRIVILEGE                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Web pods only talk to API pods                                                  │
│  • API pods only talk to DB pods                                                   │
│  • DB pods don't initiate connections                                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Network Policy Components

### Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
spec:
  # 1. Which pods does this policy apply to?
  podSelector:
    matchLabels:
      app: db
  
  # 2. What type of traffic? (Ingress, Egress, or both)
  policyTypes:
  - Ingress
  - Egress
  
  # 3. Ingress rules: Who can send traffic TO these pods?
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
  
  # 4. Egress rules: Where can these pods send traffic TO?
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: cache
    ports:
    - protocol: TCP
      port: 6379
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         NETWORK POLICY COMPONENTS                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         NetworkPolicy                                       │   │
│  │                                                                             │   │
│  │  podSelector: ───────────────────► Which pods this policy applies to       │   │
│  │    matchLabels:                                                             │   │
│  │      app: db                       (target pods)                            │   │
│  │                                                                             │   │
│  │  policyTypes: ───────────────────► What traffic to control                 │   │
│  │  - Ingress                         • Ingress = incoming                     │   │
│  │  - Egress                          • Egress = outgoing                      │   │
│  │                                                                             │   │
│  │  ingress: ───────────────────────► Rules for incoming traffic              │   │
│  │  - from:                           • from: source pods/namespaces/IPs       │   │
│  │    - podSelector: ...              • ports: allowed ports                   │   │
│  │    ports:                                                                   │   │
│  │    - port: 5432                                                             │   │
│  │                                                                             │   │
│  │  egress: ────────────────────────► Rules for outgoing traffic              │   │
│  │  - to:                             • to: destination pods/namespaces/IPs    │   │
│  │    - podSelector: ...              • ports: allowed ports                   │   │
│  │    ports:                                                                   │   │
│  │    - port: 6379                                                             │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔓 Default Behavior

### Without Network Policies

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DEFAULT: ALLOW ALL                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  By default, Kubernetes allows ALL traffic between pods:                            │
│                                                                                      │
│  • Any pod can send traffic to any other pod                                       │
│  • Any pod can receive traffic from any other pod                                  │
│  • Cross-namespace communication is allowed                                        │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          Namespace: default                                 │   │
│  │                                                                             │   │
│  │    ┌─────┐ ◄─────────────────────────────────────────────► ┌─────┐         │   │
│  │    │Pod A│                    ✅ ALLOWED                   │Pod B│         │   │
│  │    └─────┘ ◄─────────────────────────────────────────────► └─────┘         │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                              ▲                                                      │
│                              │ ✅ ALLOWED                                           │
│                              ▼                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          Namespace: production                              │   │
│  │                                                                             │   │
│  │    ┌─────┐                                                                  │   │
│  │    │Pod C│                                                                  │   │
│  │    └─────┘                                                                  │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### With Network Policy Applied

When a NetworkPolicy selects a pod:
- **Ingress**: All ingress blocked EXCEPT what rules allow
- **Egress**: All egress blocked EXCEPT what rules allow

---

## 🛠️ Creating Network Policies

### Basic Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: db                    # Apply to pods with app=db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api               # Allow from pods with app=api
    ports:
    - protocol: TCP
      port: 5432                 # On port 5432
```

### Apply and Test

```bash
# Apply policy
kubectl apply -f network-policy.yaml

# Verify
kubectl get networkpolicies
kubectl describe networkpolicy allow-api-to-db

# Test from allowed pod (should work)
kubectl exec -it api-pod -- curl db-service:5432

# Test from other pod (should fail/timeout)
kubectl exec -it web-pod -- curl db-service:5432
```

---

## 📥 Ingress Rules

### Ingress: Control Incoming Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress-policy
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  # Rule 1: Allow from specific pods
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
  
  # Rule 2: Allow from specific namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
  
  # Rule 3: Allow from IP range
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
    ports:
    - protocol: TCP
      port: 5432
```

### Ingress Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         INGRESS RULES                                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│                    Who can send traffic TO the selected pods?                       │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │   ┌─────────────────┐                                                       │   │
│  │   │  api-pod        │                                                       │   │
│  │   │  app: api       │─────────────────────────┐                             │   │
│  │   └─────────────────┘                         │                             │   │
│  │                                               │                             │   │
│  │   ┌─────────────────┐                         │                             │   │
│  │   │  monitoring     │                         ▼                             │   │
│  │   │  namespace      │────────────────────► ┌─────────────────┐              │   │
│  │   └─────────────────┘                      │   db-pod        │              │   │
│  │                                            │   app: db       │              │   │
│  │   ┌─────────────────┐                      │   port: 5432    │              │   │
│  │   │  10.0.0.0/8     │────────────────────► │                 │              │   │
│  │   │  (except .1.0)  │                      │  (protected)    │              │   │
│  │   └─────────────────┘                      └─────────────────┘              │   │
│  │                                               ▲                             │   │
│  │   ┌─────────────────┐                         │                             │   │
│  │   │  random-pod     │─────────────────────────┘                             │   │
│  │   │  (no match)     │       ❌ BLOCKED!                                     │   │
│  │   └─────────────────┘                                                       │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📤 Egress Rules

### Egress: Control Outgoing Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  # Rule 1: Allow to database
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
  
  # Rule 2: Allow to cache
  - to:
    - podSelector:
        matchLabels:
          app: cache
    ports:
    - protocol: TCP
      port: 6379
  
  # Rule 3: Allow DNS (IMPORTANT!)
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Egress Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         EGRESS RULES                                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│              Where can the selected pods send traffic TO?                           │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │                               ┌─────────────────┐                           │   │
│  │                         ┌────►│  db-pod         │  ✅ Allowed               │   │
│  │                         │     │  app: db        │                           │   │
│  │   ┌─────────────────┐   │     └─────────────────┘                           │   │
│  │   │  api-pod        │───┤                                                   │   │
│  │   │  app: api       │   │     ┌─────────────────┐                           │   │
│  │   │                 │   ├────►│  cache-pod      │  ✅ Allowed               │   │
│  │   │  (controlled)   │   │     │  app: cache     │                           │   │
│  │   └─────────────────┘   │     └─────────────────┘                           │   │
│  │                         │                                                   │   │
│  │                         │     ┌─────────────────┐                           │   │
│  │                         ├────►│  kube-dns       │  ✅ Allowed (port 53)     │   │
│  │                         │     │  (DNS)          │                           │   │
│  │                         │     └─────────────────┘                           │   │
│  │                         │                                                   │   │
│  │                         │     ┌─────────────────┐                           │   │
│  │                         └────►│  external       │  ❌ BLOCKED               │   │
│  │                               │  internet       │                           │   │
│  │                               └─────────────────┘                           │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  ⚠️  IMPORTANT: Always allow DNS (port 53) for egress policies!                    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏷️ Selectors

### Three Types of Selectors

```yaml
spec:
  ingress:
  - from:
    # 1. podSelector: Select pods in SAME namespace
    - podSelector:
        matchLabels:
          app: api
    
    # 2. namespaceSelector: Select ALL pods in matching namespaces
    - namespaceSelector:
        matchLabels:
          name: frontend
    
    # 3. ipBlock: Select by IP range
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
```

### Combining Selectors (AND vs OR)

```yaml
# OR: Separate list items (either condition)
ingress:
- from:
  - podSelector:          # Condition 1
      matchLabels:
        app: web
  - namespaceSelector:    # OR Condition 2
      matchLabels:
        name: frontend

# AND: Same list item (both conditions)
ingress:
- from:
  - podSelector:          # Condition 1
      matchLabels:
        app: web
    namespaceSelector:    # AND Condition 2 (same item!)
      matchLabels:
        name: frontend
```

### Selector Behavior

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SELECTOR COMBINATIONS                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  OR (separate items):                                                               │
│  - from:                                                                            │
│    - podSelector: {app: web}        # Pods with app=web in same namespace          │
│    - namespaceSelector: {name: x}   # OR any pod in namespace x                    │
│                                                                                      │
│  AND (same item):                                                                   │
│  - from:                                                                            │
│    - podSelector: {app: web}        # Pods with app=web                            │
│      namespaceSelector: {name: x}   # AND in namespace x                           │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Special cases:                                                                     │
│                                                                                      │
│  - podSelector: {}         # All pods in same namespace                            │
│  - namespaceSelector: {}   # All namespaces                                        │
│  - {}                      # All pods in all namespaces                            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏠 Namespace Isolation

### Isolate Entire Namespace

```yaml
# Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}           # Apply to ALL pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

### Allow Cross-Namespace Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring      # Namespace must have this label!
    ports:
    - protocol: TCP
      port: 9090
```

```bash
# Label the namespace first!
kubectl label namespace monitoring name=monitoring
```

---

## 🚫 Default Deny Policies

### Default Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Default Deny All (Ingress + Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow All (Explicit)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}                      # Empty rule = allow all
```

---

## 📋 Common Patterns

### 1. Web → API → DB Pattern

```yaml
# Allow web to API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-web
spec:
  podSelector:
    matchLabels:
      tier: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - port: 8080

---

# Allow API to DB
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-api
spec:
  podSelector:
    matchLabels:
      tier: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - port: 5432
```

### 2. Allow Same Namespace Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}       # Any pod in same namespace
```

### 3. Allow External DNS

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

## 🔌 CNI Requirements

### Network Policy Support

> ⚠️ **Important**: NetworkPolicy requires a CNI plugin that supports it!

| CNI Plugin | Network Policy Support |
|------------|------------------------|
| **Calico** | ✅ Full support |
| **Cilium** | ✅ Full support + extended |
| **Weave Net** | ✅ Full support |
| **Flannel** | ❌ No support |
| **AWS VPC CNI** | ✅ Via Calico addon |
| **Azure CNI** | ✅ With Azure NPM |
| **GKE** | ✅ Built-in |

```bash
# Check if CNI supports Network Policies
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"
```

---

## 🔧 Operations

### Common Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl apply -f network-policy.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get networkpolicies
kubectl get netpol                  # Short form
kubectl get netpol -n production

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe networkpolicy my-policy

# ═══════════════════════════════════════════════════════════════════
# TEST CONNECTIVITY
# ═══════════════════════════════════════════════════════════════════
# From allowed pod
kubectl exec -it allowed-pod -- curl target-service:80

# From blocked pod (should timeout)
kubectl exec -it blocked-pod -- curl --connect-timeout 5 target-service:80

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete networkpolicy my-policy
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Policy not working | CNI doesn't support | Use Calico/Cilium |
| All traffic blocked | Missing DNS egress | Add DNS egress rule |
| Cross-namespace blocked | Missing namespace label | Label namespace |
| Pods can't resolve DNS | Egress blocks DNS | Allow port 53 UDP |

### Debug Steps

```bash
# 1. Check CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep -E "calico|cilium"

# 2. List all network policies
kubectl get netpol -A

# 3. Describe policy
kubectl describe netpol my-policy

# 4. Check pod labels match
kubectl get pods --show-labels

# 5. Check namespace labels
kubectl get namespaces --show-labels

# 6. Test connectivity
kubectl exec -it test-pod -- nc -zv target-service 80
```

---

## 🎓 CKA Exam Tips

### Quick Network Policy Creation

```bash
# No direct kubectl create for NetworkPolicy
# Write YAML or use this template:

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
EOF
```

### Key Points for Exam

1. **podSelector: {}** = all pods in namespace
2. **namespaceSelector: {}** = all namespaces
3. **Empty ingress/egress** = deny all
4. **Empty rule {}** = allow all
5. **Remember DNS** for egress policies (port 53)
6. **Policies are additive** (union of all matching)

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Network Policy** | Firewall rules for pods |
| **podSelector** | Which pods policy applies to |
| **Ingress** | Incoming traffic rules |
| **Egress** | Outgoing traffic rules |
| **Default behavior** | Allow all (no policies) |
| **With policy** | Deny all except allowed |

### Default Deny Quick Reference

```yaml
# Deny all ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]

# Deny all egress  
spec:
  podSelector: {}
  policyTypes: [Egress]

# Deny both
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

---

## 🔜 What's Next

In **Chapter 15: DNS in Kubernetes**, we'll cover:

- CoreDNS architecture
- Service discovery via DNS
- Pod DNS configuration
- Custom DNS entries
- DNS debugging

---

*Network Policies are essential for production security - always use them!*

