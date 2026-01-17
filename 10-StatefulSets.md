# ☸️ Chapter 10: StatefulSets

> Managing stateful applications with stable identities, persistent storage, and ordered operations.

---

## 📚 Table of Contents

1. [What is a StatefulSet?](#-what-is-a-statefulset)
2. [Why StatefulSets?](#-why-statefulsets)
3. [StatefulSet vs Deployment](#-statefulset-vs-deployment)
4. [StatefulSet Components](#-statefulset-components)
5. [Stable Network Identity](#-stable-network-identity)
6. [Stable Storage](#-stable-storage)
7. [Ordered Operations](#-ordered-operations)
8. [Creating StatefulSets](#-creating-statefulsets)
9. [Headless Services](#-headless-services)
10. [Scaling StatefulSets](#-scaling-statefulsets)
11. [Update Strategies](#-update-strategies)
12. [Common Use Cases](#-common-use-cases)
13. [Operations](#-operations)
14. [Troubleshooting](#-troubleshooting)
15. [CKA Exam Tips](#-cka-exam-tips)
16. [Summary](#-summary)

---

## 📖 What is a StatefulSet?

### Definition

> **StatefulSet** is a Kubernetes workload API object that manages stateful applications. It provides guarantees about the ordering and uniqueness of Pods, with stable network identities and persistent storage.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS A STATEFULSET?                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  StatefulSet = Stable Identity + Persistent Storage + Ordered Operations            │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐│
│  │                        StatefulSet: mysql                                      ││
│  │                                                                                ││
│  │  Pod Names are PREDICTABLE and STABLE:                                        ││
│  │                                                                                ││
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                ││
│  │  │   mysql-0       │  │   mysql-1       │  │   mysql-2       │                ││
│  │  │   (master)      │  │   (replica)     │  │   (replica)     │                ││
│  │  │                 │  │                 │  │                 │                ││
│  │  │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │                ││
│  │  │  │  PVC-0    │  │  │  │  PVC-1    │  │  │  │  PVC-2    │  │                ││
│  │  │  │  10Gi     │  │  │  │  10Gi     │  │  │  │  10Gi     │  │                ││
│  │  │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │                ││
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘                ││
│  │         │                    │                    │                            ││
│  │         ▼                    ▼                    ▼                            ││
│  │  ┌───────────┐        ┌───────────┐        ┌───────────┐                      ││
│  │  │   PV-0    │        │   PV-1    │        │   PV-2    │                      ││
│  │  │  (disk)   │        │  (disk)   │        │  (disk)   │                      ││
│  │  └───────────┘        └───────────┘        └───────────┘                      ││
│  │                                                                                ││
│  │  Each pod gets its own persistent volume that follows it!                     ││
│  │                                                                                ││
│  └────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### StatefulSet Guarantees

| Guarantee | Description |
|-----------|-------------|
| **Stable Network ID** | Pods get predictable names: `<statefulset>-<ordinal>` |
| **Stable Storage** | Each pod gets its own PersistentVolumeClaim |
| **Ordered Deployment** | Pods created in order: 0, 1, 2, ... |
| **Ordered Termination** | Pods deleted in reverse order: ..., 2, 1, 0 |
| **Ordered Updates** | Pods updated in reverse order |

---

## ❓ Why StatefulSets?

### The Problem with Deployments for Stateful Apps

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                   WHY DEPLOYMENTS DON'T WORK FOR DATABASES                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  PROBLEM 1: Random Pod Names                                                        │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Deployment pods:                     StatefulSet pods:                             │
│  mysql-7b9f8c6d5-xj4k2               mysql-0                                        │
│  mysql-7b9f8c6d5-m2np9               mysql-1                                        │
│  mysql-7b9f8c6d5-q8rt5               mysql-2                                        │
│        ↑                                    ↑                                        │
│   Random suffix!                       Predictable!                                 │
│   Changes on restart!                  Same after restart!                          │
│                                                                                      │
│  Database replication needs to know: "mysql-0 is master, mysql-1,2 are replicas"   │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  PROBLEM 2: Shared Storage                                                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Deployment with PVC:                 StatefulSet with PVC:                         │
│  ┌─────┐ ┌─────┐ ┌─────┐             ┌─────┐ ┌─────┐ ┌─────┐                        │
│  │Pod 1│ │Pod 2│ │Pod 3│             │Pod 0│ │Pod 1│ │Pod 2│                        │
│  └──┬──┘ └──┬──┘ └──┬──┘             └──┬──┘ └──┬──┘ └──┬──┘                        │
│     │       │       │                   │       │       │                            │
│     └───────┼───────┘                   ▼       ▼       ▼                            │
│             ▼                        ┌─────┐ ┌─────┐ ┌─────┐                        │
│          ┌─────┐                     │PVC-0│ │PVC-1│ │PVC-2│                        │
│          │ PVC │ (shared!)           └─────┘ └─────┘ └─────┘                        │
│          └─────┘                        ↑       ↑       ↑                            │
│             ↑                         Own!    Own!    Own!                          │
│         CONFLICT!                                                                   │
│   Multiple pods writing                Each pod has its own storage                 │
│   to same storage!                                                                  │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  PROBLEM 3: No Ordering                                                             │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Database cluster startup:                                                          │
│  1. Master must start FIRST                                                         │
│  2. Replicas start AFTER master is ready                                           │
│  3. Replicas need to know master's address                                         │
│                                                                                      │
│  Deployment: All pods start simultaneously (chaos!)                                 │
│  StatefulSet: Pods start in order (0 → 1 → 2)                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 StatefulSet vs Deployment

### Comparison Table

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         STATEFULSET VS DEPLOYMENT                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Feature              │ Deployment              │ StatefulSet                       │
│  ─────────────────────┼─────────────────────────┼───────────────────────────────────│
│  Pod Names            │ Random suffix           │ Ordered index (0, 1, 2...)        │
│  Pod Identity         │ Interchangeable         │ Unique and stable                 │
│  DNS                  │ Via Service only        │ Individual pod DNS names          │
│  Storage              │ Shared PVC              │ PVC per pod                       │
│  Startup Order        │ Parallel                │ Sequential (0→1→2)                │
│  Shutdown Order       │ Parallel                │ Reverse sequential (2→1→0)        │
│  Scaling Up           │ All at once             │ One at a time, in order           │
│  Scaling Down         │ All at once             │ One at a time, reverse order      │
│  Use Case             │ Stateless apps          │ Databases, message queues         │
│  Service Type         │ ClusterIP/NodePort/LB   │ Headless Service required         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### When to Use What

| Application Type | Use |
|------------------|-----|
| Web servers | Deployment |
| APIs | Deployment |
| Frontend apps | Deployment |
| MySQL/PostgreSQL | StatefulSet |
| MongoDB | StatefulSet |
| Redis (cluster) | StatefulSet |
| Kafka | StatefulSet |
| Elasticsearch | StatefulSet |
| ZooKeeper | StatefulSet |
| etcd | StatefulSet |

---

## 🧩 StatefulSet Components

### Required Components

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         STATEFULSET COMPONENTS                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. Headless Service (REQUIRED)                                                     │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  apiVersion: v1                                                                     │
│  kind: Service                                                                      │
│  metadata:                                                                          │
│    name: mysql-headless                                                             │
│  spec:                                                                              │
│    clusterIP: None          ◄─── Makes it headless!                                │
│    selector:                                                                        │
│      app: mysql                                                                     │
│    ports:                                                                           │
│    - port: 3306                                                                     │
│                                                                                      │
│  2. StatefulSet                                                                     │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  apiVersion: apps/v1                                                                │
│  kind: StatefulSet                                                                  │
│  spec:                                                                              │
│    serviceName: mysql-headless   ◄─── References headless service                  │
│    replicas: 3                                                                      │
│    ...                                                                              │
│                                                                                      │
│  3. StorageClass (for dynamic provisioning)                                        │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  apiVersion: storage.k8s.io/v1                                                      │
│  kind: StorageClass                                                                 │
│  metadata:                                                                          │
│    name: fast-storage                                                               │
│  provisioner: kubernetes.io/aws-ebs                                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🌐 Stable Network Identity

### Pod DNS Names

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         STATEFULSET DNS NAMES                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  StatefulSet: mysql                                                                 │
│  Namespace: default                                                                 │
│  Headless Service: mysql-headless                                                   │
│                                                                                      │
│  Pod Names:                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  mysql-0                                                                    │   │
│  │  mysql-1                                                                    │   │
│  │  mysql-2                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  DNS Names (from within cluster):                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │  Short form:                                                                │   │
│  │  mysql-0.mysql-headless                                                     │   │
│  │  mysql-1.mysql-headless                                                     │   │
│  │  mysql-2.mysql-headless                                                     │   │
│  │                                                                             │   │
│  │  Full FQDN:                                                                 │   │
│  │  mysql-0.mysql-headless.default.svc.cluster.local                          │   │
│  │  mysql-1.mysql-headless.default.svc.cluster.local                          │   │
│  │  mysql-2.mysql-headless.default.svc.cluster.local                          │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Format: <pod-name>.<service-name>.<namespace>.svc.cluster.local                   │
│                                                                                      │
│  Example: Connect to master                                                         │
│  mysql -h mysql-0.mysql-headless -u root -p                                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Network Identity Persistence

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      IDENTITY SURVIVES POD RESTART                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Before restart:                    After restart:                                  │
│                                                                                      │
│  mysql-0 (IP: 10.244.1.5)          mysql-0 (IP: 10.244.1.99)  ← IP may change      │
│  mysql-1 (IP: 10.244.2.8)          mysql-1 (IP: 10.244.2.8)                         │
│  mysql-2 (IP: 10.244.3.2)          mysql-2 (IP: 10.244.3.2)                         │
│                                                                                      │
│  But DNS name stays the same:                                                       │
│  mysql-0.mysql-headless → Always resolves to mysql-0's current IP                  │
│                                                                                      │
│  ✅ Applications connect by DNS name, not IP!                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 💾 Stable Storage

### VolumeClaimTemplates

```yaml
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-storage"
      resources:
        requests:
          storage: 10Gi
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         VOLUMECLAIMTEMPLATES                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  StatefulSet creates PVCs automatically for each pod:                               │
│                                                                                      │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐           │
│  │     mysql-0       │    │     mysql-1       │    │     mysql-2       │           │
│  │                   │    │                   │    │                   │           │
│  │  /var/lib/mysql   │    │  /var/lib/mysql   │    │  /var/lib/mysql   │           │
│  │       │           │    │       │           │    │       │           │           │
│  └───────┼───────────┘    └───────┼───────────┘    └───────┼───────────┘           │
│          │                        │                        │                        │
│          ▼                        ▼                        ▼                        │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐           │
│  │  data-mysql-0     │    │  data-mysql-1     │    │  data-mysql-2     │           │
│  │  (PVC)            │    │  (PVC)            │    │  (PVC)            │           │
│  └───────────────────┘    └───────────────────┘    └───────────────────┘           │
│          │                        │                        │                        │
│          ▼                        ▼                        ▼                        │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐           │
│  │  PersistentVolume │    │  PersistentVolume │    │  PersistentVolume │           │
│  │  (10Gi SSD)       │    │  (10Gi SSD)       │    │  (10Gi SSD)       │           │
│  └───────────────────┘    └───────────────────┘    └───────────────────┘           │
│                                                                                      │
│  PVC naming: <volumeClaimTemplate-name>-<statefulset-name>-<ordinal>               │
│  Example: data-mysql-0, data-mysql-1, data-mysql-2                                 │
│                                                                                      │
│  ⚠️  IMPORTANT: PVCs are NOT deleted when StatefulSet is deleted!                  │
│      (Data is preserved for recovery)                                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Ordered Operations

### Deployment Order

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ORDERED OPERATIONS                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  CREATION (Scale from 0 → 3)                                                        │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Time ──────────────────────────────────────────────────────────────────────────►   │
│                                                                                      │
│  Step 1: Create mysql-0                                                             │
│          Wait until Running and Ready                                               │
│                                                                                      │
│  Step 2: Create mysql-1                                                             │
│          Wait until Running and Ready                                               │
│                                                                                      │
│  Step 3: Create mysql-2                                                             │
│          Wait until Running and Ready                                               │
│                                                                                      │
│  ┌───────────────┐                                                                  │
│  │   mysql-0     │ ════════════════════════════════════════════════════►           │
│  │   (master)    │                                                                  │
│  └───────────────┘                                                                  │
│                   ┌───────────────┐                                                 │
│                   │   mysql-1     │ ═══════════════════════════════════►           │
│                   │   (replica)   │                                                 │
│                   └───────────────┘                                                 │
│                                    ┌───────────────┐                                │
│                                    │   mysql-2     │ ══════════════════►           │
│                                    │   (replica)   │                                │
│                                    └───────────────┘                                │
│                                                                                      │
│  DELETION (Scale from 3 → 0)                                                        │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Step 1: Delete mysql-2 (highest index first)                                      │
│  Step 2: Delete mysql-1                                                             │
│  Step 3: Delete mysql-0 (master last)                                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Pod Management Policy

```yaml
spec:
  # Default: OrderedReady
  podManagementPolicy: OrderedReady   # Sequential
  
  # Alternative: Parallel
  podManagementPolicy: Parallel       # All at once (faster, less safe)
```

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `OrderedReady` | Sequential, waits for Ready | Databases, ordered systems |
| `Parallel` | All pods at once | Faster startup, non-ordered |

---

## 🛠️ Creating StatefulSets

### Complete Example: MySQL Cluster

```yaml
# 1. Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None           # Headless!
  selector:
    app: mysql

---

# 2. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless    # Reference to headless service
  replicas: 3
  
  selector:
    matchLabels:
      app: mysql
  
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secrets
              key: root-password
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -e
            - "SELECT 1"
          initialDelaySeconds: 5
          periodSeconds: 5
  
  # Volume claim template - creates PVC per pod
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

---

## 🔊 Headless Services

### What is a Headless Service?

> A **Headless Service** is a Service with `clusterIP: None`. Instead of providing a single virtual IP, it returns the IPs of all backing pods directly.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    REGULAR SERVICE VS HEADLESS SERVICE                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Regular Service (clusterIP: 10.96.0.100)                                           │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  DNS Query: mysql-service → 10.96.0.100 (single virtual IP)                        │
│                                                                                      │
│  Client ──► Service (10.96.0.100) ──► Load balanced to pods                        │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Headless Service (clusterIP: None)                                                 │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  DNS Query: mysql-headless → Returns ALL pod IPs:                                   │
│             10.244.1.5 (mysql-0)                                                    │
│             10.244.2.8 (mysql-1)                                                    │
│             10.244.3.2 (mysql-2)                                                    │
│                                                                                      │
│  DNS Query: mysql-0.mysql-headless → 10.244.1.5 (specific pod!)                    │
│                                                                                      │
│  Client ──► Direct connection to specific pod                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Why Headless for StatefulSets?

| Reason | Explanation |
|--------|-------------|
| **Individual pod access** | Connect to specific pod (e.g., master) |
| **Client-side load balancing** | Application decides which pod |
| **Discovery** | Get all pod IPs for cluster configuration |
| **Replication** | Replicas need to find master by DNS |

---

## 📈 Scaling StatefulSets

### Scaling Commands

```bash
# Scale up
kubectl scale statefulset mysql --replicas=5

# Scale down
kubectl scale statefulset mysql --replicas=2

# Edit and change replicas
kubectl edit statefulset mysql
```

### Scaling Behavior

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         STATEFULSET SCALING                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  SCALE UP: 3 → 5                                                                    │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Existing:    mysql-0   mysql-1   mysql-2                                           │
│  Created:                                     mysql-3   mysql-4                     │
│                                                  │          │                        │
│                                      (wait for 3)          │                        │
│                                                    (wait for 4)                     │
│                                                                                      │
│  SCALE DOWN: 5 → 3                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Before: mysql-0   mysql-1   mysql-2   mysql-3   mysql-4                           │
│                                                      │         │                     │
│                                           (delete 4 first)     │                     │
│                                                    (then delete 3)                  │
│  After:  mysql-0   mysql-1   mysql-2                                                │
│                                                                                      │
│  ⚠️  PVCs are NOT deleted during scale down!                                        │
│      data-mysql-3 and data-mysql-4 still exist                                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0          # Update all pods (0 and higher)
```

### Partition Updates

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2          # Only update pods >= 2 (canary)
```

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         PARTITION UPDATES (CANARY)                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  partition: 2 means: Only update pods with ordinal >= 2                             │
│                                                                                      │
│  Before update:                                                                     │
│  mysql-0 (v1)   mysql-1 (v1)   mysql-2 (v1)   mysql-3 (v1)   mysql-4 (v1)          │
│                                                                                      │
│  After update (partition: 2):                                                       │
│  mysql-0 (v1)   mysql-1 (v1)   mysql-2 (v2)   mysql-3 (v2)   mysql-4 (v2)          │
│        ↑             ↑              ↑             ↑             ↑                   │
│   NOT updated   NOT updated    Updated      Updated      Updated                    │
│                                                                                      │
│  Test v2 on mysql-2,3,4 first, then set partition: 0 to update all                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete    # Only update when pod is manually deleted
```

---

## 💼 Common Use Cases

### 1. Database Clusters

```yaml
# PostgreSQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  serviceName: postgresql-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

### 2. Message Queues (Kafka)

```yaml
# Kafka StatefulSet snippet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-headless
  replicas: 3
  template:
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.0.0
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name  # Uses ordinal
```

---

## 🔧 Operations

### Common Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl apply -f statefulset.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get statefulsets
kubectl get sts                          # Short form
kubectl get sts -o wide

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe sts mysql

# ═══════════════════════════════════════════════════════════════════
# VIEW PODS (ordered names!)
# ═══════════════════════════════════════════════════════════════════
kubectl get pods -l app=mysql

# ═══════════════════════════════════════════════════════════════════
# VIEW PVCs
# ═══════════════════════════════════════════════════════════════════
kubectl get pvc -l app=mysql

# ═══════════════════════════════════════════════════════════════════
# SCALE
# ═══════════════════════════════════════════════════════════════════
kubectl scale sts mysql --replicas=5

# ═══════════════════════════════════════════════════════════════════
# UPDATE
# ═══════════════════════════════════════════════════════════════════
kubectl set image sts/mysql mysql=mysql:8.1
kubectl rollout status sts/mysql

# ═══════════════════════════════════════════════════════════════════
# DELETE (PVCs remain!)
# ═══════════════════════════════════════════════════════════════════
kubectl delete sts mysql
kubectl delete pvc -l app=mysql          # Delete PVCs manually
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Pod stuck Pending | No PV available | Check StorageClass, provision PV |
| Pod not Ready | Readiness probe failing | Check probe, app startup |
| DNS not resolving | Headless service wrong | Check clusterIP: None |
| Data not persisting | PVC not mounted | Check volumeClaimTemplates |

### Debug Commands

```bash
# Check StatefulSet status
kubectl describe sts mysql

# Check pods
kubectl get pods -l app=mysql -o wide

# Check PVCs
kubectl get pvc

# Check PVs
kubectl get pv

# Check headless service
kubectl get svc mysql-headless

# DNS resolution test
kubectl run -it --rm debug --image=busybox -- nslookup mysql-0.mysql-headless

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

---

## 🎓 CKA Exam Tips

### Quick StatefulSet Creation

```bash
# Generate template from deployment
kubectl create deployment mysql --image=mysql:8.0 --dry-run=client -o yaml > sts.yaml
# Then modify: kind to StatefulSet, add serviceName, volumeClaimTemplates

# Don't forget headless service!
kubectl create service clusterip mysql-headless --tcp=3306:3306 --dry-run=client -o yaml | \
  sed 's/clusterIP: .*/clusterIP: None/' > svc.yaml
```

### Key Points for Exam

1. **serviceName is required** - must reference a headless service
2. **Headless = clusterIP: None** - remember this
3. **volumeClaimTemplates** - for per-pod storage
4. **Pod names are predictable** - `<name>-0`, `<name>-1`, etc.
5. **PVCs persist after delete** - manual cleanup needed

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **StatefulSet** | Workload for stateful applications |
| **Stable Identity** | Predictable pod names (mysql-0, mysql-1) |
| **Stable Storage** | PVC per pod via volumeClaimTemplates |
| **Ordered Operations** | Sequential create/delete/update |
| **Headless Service** | Required, clusterIP: None |

### StatefulSet Requirements

1. Headless Service (clusterIP: None)
2. serviceName in StatefulSet spec
3. volumeClaimTemplates for storage
4. StorageClass for dynamic provisioning

### Essential Commands

```bash
kubectl get sts
kubectl describe sts <name>
kubectl scale sts <name> --replicas=<n>
kubectl rollout status sts/<name>
```

---

## 🔜 What's Next

In **Chapter 11: Jobs & CronJobs**, we'll cover:

- Running batch processing workloads
- Job completion and parallelism
- Scheduled tasks with CronJobs
- Handling job failures

---

*StatefulSets are essential for databases and distributed systems!*

