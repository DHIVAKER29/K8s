# ☸️ Chapter 09: DaemonSets

> Ensuring a pod runs on every node - essential for cluster-wide services like logging, monitoring, and networking.

---

## 📚 Table of Contents

1. [What is a DaemonSet?](#-what-is-a-daemonset)
2. [Why DaemonSets?](#-why-daemonsets)
3. [DaemonSet vs Deployment](#-daemonset-vs-deployment)
4. [DaemonSet Architecture](#-daemonset-architecture)
5. [Creating DaemonSets](#-creating-daemonsets)
6. [DaemonSet Manifest](#-daemonset-manifest)
7. [Node Selection](#-node-selection)
8. [Tolerations](#-tolerations)
9. [Update Strategies](#-update-strategies)
10. [Common Use Cases](#-common-use-cases)
11. [Operations](#-operations)
12. [Troubleshooting](#-troubleshooting)
13. [CKA Exam Tips](#-cka-exam-tips)
14. [Summary](#-summary)

---

## 📖 What is a DaemonSet?

### Definition

> **DaemonSet** ensures that all (or some) nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed, those Pods are garbage collected.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS A DAEMONSET?                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  "Run exactly ONE copy of this pod on EVERY node in the cluster"                    │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │                         Kubernetes Cluster                                     ││
│  │                                                                                ││
│  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐    ││
│  │  │   Node 1            │  │   Node 2            │  │   Node 3            │    ││
│  │  │                     │  │                     │  │                     │    ││
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │    ││
│  │  │  │ DaemonSet Pod │  │  │  │ DaemonSet Pod │  │  │  │ DaemonSet Pod │  │    ││
│  │  │  │ (fluentd)     │  │  │  │ (fluentd)     │  │  │  │ (fluentd)     │  │    ││
│  │  │  └───────────────┘  │  │  └───────────────┘  │  │  └───────────────┘  │    ││
│  │  │                     │  │                     │  │                     │    ││
│  │  │  [Other pods...]    │  │  [Other pods...]    │  │  [Other pods...]    │    ││
│  │  │                     │  │                     │  │                     │    ││
│  │  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘    ││
│  │                                                                                ││
│  │  If Node 4 is added → DaemonSet automatically creates a pod on it            ││
│  │  If Node 2 is removed → DaemonSet pod is garbage collected                   ││
│  │                                                                                ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### DaemonSet Characteristics

| Characteristic | Description |
|----------------|-------------|
| **One pod per node** | Exactly one replica on each eligible node |
| **Automatic scaling** | Pods added/removed as nodes join/leave |
| **Node-aware** | Can target specific nodes with selectors |
| **System services** | Perfect for infrastructure workloads |

---

## ❓ Why DaemonSets?

### Use Cases

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DAEMONSET USE CASES                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. LOG COLLECTION                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                          │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                                          │
│  │          │  │          │  │          │                                          │
│  │ [fluentd]│  │ [fluentd]│  │ [fluentd]│  → Centralized Logging                  │
│  │  ↑ logs  │  │  ↑ logs  │  │  ↑ logs  │     (Elasticsearch, Splunk)             │
│  │[app pods]│  │[app pods]│  │[app pods]│                                          │
│  └──────────┘  └──────────┘  └──────────┘                                          │
│                                                                                      │
│  Tools: Fluentd, Fluent Bit, Filebeat, Logstash                                    │
│                                                                                      │
│  2. MONITORING / METRICS                                                            │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                          │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                                          │
│  │          │  │          │  │          │                                          │
│  │[node-exp]│  │[node-exp]│  │[node-exp]│  → Prometheus                            │
│  │  metrics │  │  metrics │  │  metrics │     (scrapes all exporters)              │
│  └──────────┘  └──────────┘  └──────────┘                                          │
│                                                                                      │
│  Tools: Node Exporter, Datadog Agent, New Relic                                    │
│                                                                                      │
│  3. NETWORK PLUGINS (CNI)                                                           │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                          │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                                          │
│  │          │  │          │  │          │                                          │
│  │ [calico] │  │ [calico] │  │ [calico] │  → Pod Networking                        │
│  │ [weave]  │  │ [weave]  │  │ [weave]  │                                          │
│  └──────────┘  └──────────┘  └──────────┘                                          │
│                                                                                      │
│  Tools: Calico, Weave, Flannel, Cilium                                             │
│                                                                                      │
│  4. STORAGE PLUGINS                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                          │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                                          │
│  │          │  │          │  │          │                                          │
│  │[csi-node]│  │[csi-node]│  │[csi-node]│  → Storage Backend                       │
│  │          │  │          │  │          │     (AWS EBS, GCP PD, Ceph)              │
│  └──────────┘  └──────────┘  └──────────┘                                          │
│                                                                                      │
│  Tools: CSI drivers (AWS EBS, Azure Disk, Ceph)                                    │
│                                                                                      │
│  5. SECURITY AGENTS                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Tools: Falco, Sysdig, Twistlock, Aqua Security                                    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 DaemonSet vs Deployment

### Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DAEMONSET VS DEPLOYMENT                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Feature             │ DaemonSet              │ Deployment                          │
│  ────────────────────┼────────────────────────┼─────────────────────────────────────│
│  Replicas            │ One per node           │ Specified number                    │
│  Scaling             │ Automatic with nodes   │ Manual or HPA                       │
│  Scheduling          │ Node-aware             │ Resource-aware                      │
│  Use case            │ System services        │ Application workloads               │
│  Pod placement       │ Guaranteed on nodes    │ Scheduler decides                   │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Deployment (replicas: 3)            DaemonSet                                      │
│                                                                                      │
│  ┌────────┐ ┌────────┐ ┌────────┐   ┌────────┐ ┌────────┐ ┌────────┐              │
│  │ Node 1 │ │ Node 2 │ │ Node 3 │   │ Node 1 │ │ Node 2 │ │ Node 3 │              │
│  │        │ │        │ │        │   │        │ │        │ │        │              │
│  │ [Pod]  │ │ [Pod]  │ │        │   │ [Pod]  │ │ [Pod]  │ │ [Pod]  │              │
│  │ [Pod]  │ │        │ │        │   │        │ │        │ │        │              │
│  └────────┘ └────────┘ └────────┘   └────────┘ └────────┘ └────────┘              │
│                                                                                      │
│  Scheduler places pods where         One pod on every node                          │
│  resources are available             (guaranteed)                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### When to Use What

| Scenario | Use |
|----------|-----|
| Web application | Deployment |
| API service | Deployment |
| Log collector on all nodes | DaemonSet |
| Monitoring agent on all nodes | DaemonSet |
| Network plugin | DaemonSet |
| GPU driver installer | DaemonSet |

---

## 🏗️ DaemonSet Architecture

### How DaemonSets Work

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DAEMONSET CONTROLLER                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                       DaemonSet Controller                                     │ │
│  │                                                                               │ │
│  │   1. Watch for node changes (add/remove)                                     │ │
│  │   2. Watch for DaemonSet changes                                             │ │
│  │   3. Ensure one pod per eligible node                                        │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                         │                                           │
│              ┌──────────────────────────┼──────────────────────────┐               │
│              │                          │                          │               │
│              ▼                          ▼                          ▼               │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │      Node 1         │  │      Node 2         │  │      Node 3         │        │
│  │                     │  │                     │  │                     │        │
│  │  DaemonSet creates  │  │  DaemonSet creates  │  │  DaemonSet creates  │        │
│  │  pod with nodeName  │  │  pod with nodeName  │  │  pod with nodeName  │        │
│  │  set to "Node 1"    │  │  set to "Node 2"    │  │  set to "Node 3"    │        │
│  │                     │  │                     │  │                     │        │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │        │
│  │  │     Pod       │  │  │  │     Pod       │  │  │  │     Pod       │  │        │
│  │  │  nodeName:    │  │  │  │  nodeName:    │  │  │  │  nodeName:    │  │        │
│  │  │   node-1      │  │  │  │   node-2      │  │  │  │   node-3      │  │        │
│  │  └───────────────┘  │  │  └───────────────┘  │  │  └───────────────┘  │        │
│  │                     │  │                     │  │                     │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
│                                                                                      │
│  DaemonSet bypasses the scheduler - sets nodeName directly                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Creating DaemonSets

### Imperative (Limited)

```bash
# No direct kubectl create daemonset command
# Must use YAML

# Generate from deployment template
kubectl create deployment fluentd --image=fluentd --dry-run=client -o yaml | \
  sed 's/Deployment/DaemonSet/g' | \
  sed '/replicas:/d' | \
  sed '/strategy:/d' > daemonset.yaml
```

### Declarative

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.14
```

```bash
kubectl apply -f daemonset.yaml
```

---

## 📝 DaemonSet Manifest

### Complete DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-daemonset
  namespace: kube-system
  labels:
    app: fluentd
    tier: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  
  # Update strategy
  updateStrategy:
    type: RollingUpdate          # or OnDelete
    rollingUpdate:
      maxUnavailable: 1          # Max pods that can be unavailable
  
  # Revision history
  revisionHistoryLimit: 10
  
  # Min ready seconds
  minReadySeconds: 0
  
  # Pod template
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      # Node selection
      nodeSelector:
        kubernetes.io/os: linux
      
      # Tolerations for running on all nodes
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      # Service account
      serviceAccountName: fluentd
      
      containers:
      - name: fluentd
        image: fluentd:v1.14
        
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "200m"
            memory: "500Mi"
        
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc
        
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
      
      terminationGracePeriodSeconds: 30
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

---

## 🎯 Node Selection

### Run on Specific Nodes

```yaml
spec:
  template:
    spec:
      # Node selector (simple)
      nodeSelector:
        disktype: ssd
        environment: production
```

### Node Affinity (Advanced)

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - worker
                - compute
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - us-east-1a
```

### Running on Subset of Nodes

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DAEMONSET NODE SELECTION                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Scenario: Run GPU driver only on GPU nodes                                         │
│                                                                                      │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │    Node 1      │  │    Node 2      │  │    Node 3      │  │    Node 4      │    │
│  │  labels:       │  │  labels:       │  │  labels:       │  │  labels:       │    │
│  │   gpu: nvidia  │  │   gpu: nvidia  │  │   (no gpu)     │  │   (no gpu)     │    │
│  │                │  │                │  │                │  │                │    │
│  │ ┌────────────┐ │  │ ┌────────────┐ │  │                │  │                │    │
│  │ │ GPU Driver │ │  │ │ GPU Driver │ │  │    (no pod)    │  │    (no pod)    │    │
│  │ │ DaemonSet  │ │  │ │ DaemonSet  │ │  │                │  │                │    │
│  │ └────────────┘ │  │ └────────────┘ │  │                │  │                │    │
│  └────────────────┘  └────────────────┘  └────────────────┘  └────────────────┘    │
│                                                                                      │
│  DaemonSet:                                                                         │
│  spec:                                                                              │
│    template:                                                                        │
│      spec:                                                                          │
│        nodeSelector:                                                                │
│          gpu: nvidia          # Only nodes with this label                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Tolerations

### Running on Control Plane Nodes

By default, control plane nodes have taints that prevent regular pods from running. DaemonSets often need to run on ALL nodes including control plane.

```yaml
spec:
  template:
    spec:
      tolerations:
      # Tolerate control-plane taint (K8s 1.24+)
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      
      # Tolerate master taint (older K8s)
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      # Tolerate all taints (run everywhere)
      - operator: Exists
```

### Common Tolerations

```yaml
tolerations:
# Tolerate specific taint
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"

# Tolerate any value for key
- key: "dedicated"
  operator: "Exists"
  effect: "NoSchedule"

# Tolerate all taints (system pods)
- operator: "Exists"
```

---

## 🔄 Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1        # Update one node at a time
```

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ROLLING UPDATE                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Step 1: Node 1 updated                                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐                                                   │
│  │  v2    │ │  v1    │ │  v1    │                                                   │
│  └────────┘ └────────┘ └────────┘                                                   │
│                                                                                      │
│  Step 2: Node 2 updated                                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐                                                   │
│  │  v2    │ │  v2    │ │  v1    │                                                   │
│  └────────┘ └────────┘ └────────┘                                                   │
│                                                                                      │
│  Step 3: Node 3 updated                                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐                                                   │
│  │  v2    │ │  v2    │ │  v2    │                                                   │
│  └────────┘ └────────┘ └────────┘                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

- Pods only updated when manually deleted
- Gives you full control over update timing
- Useful for critical system services

---

## 💼 Common Use Cases

### 1. Node Exporter (Monitoring)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
        ports:
        - containerPort: 9100
          hostPort: 9100
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

### 2. Fluent Bit (Logging)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - operator: Exists
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
```

---

## 🔧 Operations

### Common Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl apply -f daemonset.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get daemonsets
kubectl get ds                           # Short form
kubectl get ds -o wide
kubectl get ds -n kube-system            # System DaemonSets

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe ds fluentd

# ═══════════════════════════════════════════════════════════════════
# VIEW PODS
# ═══════════════════════════════════════════════════════════════════
kubectl get pods -l app=fluentd -o wide  # See which nodes

# ═══════════════════════════════════════════════════════════════════
# UPDATE
# ═══════════════════════════════════════════════════════════════════
kubectl set image ds/fluentd fluentd=fluentd:v1.15
kubectl edit ds fluentd
kubectl apply -f daemonset.yaml

# ═══════════════════════════════════════════════════════════════════
# ROLLOUT
# ═══════════════════════════════════════════════════════════════════
kubectl rollout status ds/fluentd
kubectl rollout history ds/fluentd
kubectl rollout undo ds/fluentd

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete ds fluentd
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Pod not on all nodes | Node taints | Add tolerations |
| Pod not on specific node | Node labels missing | Add nodeSelector |
| Pod pending | Resource constraints | Reduce requests |
| Pod not updating | OnDelete strategy | Delete pods manually |

### Debug Commands

```bash
# Check DaemonSet status
kubectl describe ds fluentd

# Check which nodes have pods
kubectl get pods -l app=fluentd -o wide

# Check for missing pods
kubectl get nodes
kubectl get pods -l app=fluentd -o wide | wc -l  # Compare counts

# Check node taints
kubectl describe node <node-name> | grep Taints

# Check events
kubectl get events --field-selector involvedObject.name=fluentd
```

---

## 🎓 CKA Exam Tips

### Quick DaemonSet Creation

```bash
# Create DaemonSet YAML quickly
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
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

### Common Exam Scenarios

```bash
# Scenario: Create DaemonSet on all nodes
# Remember: Add tolerations for control-plane nodes

# Scenario: Create DaemonSet on specific nodes
# Remember: Use nodeSelector

# Scenario: Update DaemonSet image
kubectl set image ds/nginx-ds nginx=nginx:1.20

# Scenario: Check rollout status
kubectl rollout status ds/nginx-ds
```

### Key Points

1. **No replicas field** - DaemonSet scales with nodes
2. **Tolerations** - needed to run on control-plane
3. **nodeSelector** - to run on subset of nodes
4. **updateStrategy** - RollingUpdate or OnDelete

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **DaemonSet** | One pod per node (or selected nodes) |
| **Use Cases** | Logging, monitoring, networking, storage |
| **nodeSelector** | Run on specific nodes |
| **Tolerations** | Run on tainted nodes (including control-plane) |
| **updateStrategy** | RollingUpdate or OnDelete |

### DaemonSet vs Deployment

| Aspect | DaemonSet | Deployment |
|--------|-----------|------------|
| Replicas | One per node | Specified number |
| Scaling | Automatic | Manual/HPA |
| Purpose | System services | Applications |

### Essential Commands

```bash
kubectl get ds
kubectl describe ds <name>
kubectl set image ds/<name> <container>=<image>
kubectl rollout status ds/<name>
```

---

## 🔜 What's Next

In **Chapter 10: StatefulSets**, we'll cover:

- Stateful applications in Kubernetes
- Stable network identities
- Ordered deployment and scaling
- Persistent storage per pod

---

*DaemonSets are essential for cluster-wide infrastructure services!*

