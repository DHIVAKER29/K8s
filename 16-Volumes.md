# Chapter 16: Volumes - Persistent Storage in Kubernetes

## Introduction

In Docker, we learned that containers are ephemeral - when they die, their data dies with them. We solved this with Docker volumes. Kubernetes faces the same challenge but at a much larger scale, with pods potentially running across multiple nodes.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE CONTAINER STORAGE PROBLEM                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Container Lifecycle:                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │
│   │  Start   │───▶│  Write   │───▶│   Die    │───▶│  Data    │        │
│   │Container │    │  Data    │    │          │    │  LOST!   │        │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘        │
│                                                                         │
│   With Volumes:                                                         │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │
│   │  Start   │───▶│  Write   │───▶│   Die    │───▶│  Data    │        │
│   │Container │    │to Volume │    │          │    │  SAFE!   │        │
│   └──────────┘    └────┬─────┘    └──────────┘    └────┬─────┘        │
│                        │                               │               │
│                        ▼                               ▼               │
│                   ┌────────────────────────────────────────┐           │
│                   │         PERSISTENT VOLUME              │           │
│                   │         (Exists independently)         │           │
│                   └────────────────────────────────────────┘           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Understanding Kubernetes Volumes

### 1.1 What is a Volume?

A **Volume** in Kubernetes is a directory, possibly with data in it, which is accessible to containers in a pod. Unlike Docker volumes, Kubernetes volumes have explicit lifetimes - they live as long as the pod that uses them.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VOLUME CONCEPT IN KUBERNETES                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                           POD                                    │   │
│   │  ┌──────────────────┐      ┌──────────────────┐                 │   │
│   │  │   Container A    │      │   Container B    │                 │   │
│   │  │                  │      │                  │                 │   │
│   │  │  /app/data ──────┼──────┼──▶ /shared/data  │                 │   │
│   │  │                  │      │                  │                 │   │
│   │  └────────┬─────────┘      └────────┬─────────┘                 │   │
│   │           │                         │                           │   │
│   │           └───────────┬─────────────┘                           │   │
│   │                       ▼                                         │   │
│   │              ┌────────────────┐                                 │   │
│   │              │    VOLUME      │                                 │   │
│   │              │   "my-data"    │                                 │   │
│   │              └────────────────┘                                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Key Points:                                                           │
│   • Volume is declared at Pod level                                     │
│   • Containers mount the volume at different paths                      │
│   • Data is shared between containers                                   │
│   • Volume lifecycle = Pod lifecycle (for most types)                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Volume vs PersistentVolume

| Aspect | Volume | PersistentVolume (PV) |
|--------|--------|----------------------|
| **Lifecycle** | Same as Pod | Independent of Pod |
| **Definition** | Inside Pod spec | Cluster-level resource |
| **Provisioning** | Defined per-pod | Pre-provisioned or dynamic |
| **Portability** | Tied to pod | Reusable across pods |
| **Use Case** | Temporary data, sharing | Database, persistent apps |

### 1.3 Docker Volume to Kubernetes Mapping

| Docker Concept | Kubernetes Equivalent |
|---------------|----------------------|
| `docker run -v mydata:/app/data` | PersistentVolumeClaim mounted in Pod |
| `docker run -v /host/path:/container/path` | hostPath volume |
| `docker run --tmpfs /app/temp` | emptyDir with `medium: Memory` |
| `docker volume create` | PersistentVolume |
| Docker volume driver | StorageClass |

---

## 2. Volume Types Overview

Kubernetes supports many volume types for different use cases:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        VOLUME TYPES TAXONOMY                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    EPHEMERAL VOLUMES                             │   │
│   │   (Lifecycle tied to Pod - data lost when Pod dies)             │   │
│   ├─────────────────────────────────────────────────────────────────┤   │
│   │  emptyDir     │  Temporary directory, starts empty              │   │
│   │  configMap    │  Inject configuration files                     │   │
│   │  secret       │  Inject sensitive data                          │   │
│   │  downwardAPI  │  Expose pod/container metadata                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    PERSISTENT VOLUMES                            │   │
│   │   (Lifecycle independent - data survives Pod restarts)          │   │
│   ├─────────────────────────────────────────────────────────────────┤   │
│   │  hostPath        │  Mount from node's filesystem                │   │
│   │  nfs             │  Network File System                         │   │
│   │  persistentVolume│  Abstracted persistent storage               │   │
│   │  csi             │  Container Storage Interface drivers         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    CLOUD PROVIDER VOLUMES                        │   │
│   │   (Managed by cloud provider)                                   │   │
│   ├─────────────────────────────────────────────────────────────────┤   │
│   │  awsElasticBlockStore  │  AWS EBS volumes                       │   │
│   │  gcePersistentDisk     │  Google Compute Engine PD              │   │
│   │  azureDisk             │  Azure managed disks                   │   │
│   │  azureFile             │  Azure file shares                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Complete Volume Types Reference

| Volume Type | Persistence | Use Case | CKA Relevance |
|-------------|-------------|----------|---------------|
| **emptyDir** | Pod lifetime | Cache, temp files, sharing between containers | 🔴 HIGH |
| **hostPath** | Node lifetime | Access node files, single-node dev | 🔴 HIGH |
| **configMap** | Updates with CM | Configuration files | 🔴 HIGH |
| **secret** | Updates with Secret | Credentials, certificates | 🔴 HIGH |
| **persistentVolumeClaim** | Cluster lifetime | Databases, stateful apps | 🔴 HIGH |
| **nfs** | External | Shared storage | 🟡 MEDIUM |
| **downwardAPI** | Pod lifetime | Pod metadata | 🟡 MEDIUM |
| **projected** | Various | Combine multiple sources | 🟡 MEDIUM |
| **csi** | External | Any storage via CSI | 🟡 MEDIUM |

---

## 3. emptyDir Volume

### 3.1 What is emptyDir?

An **emptyDir** volume is created when a Pod is assigned to a node and exists as long as the Pod runs on that node. It starts empty, hence the name.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         emptyDir LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Pod Scheduled to Node                                                 │
│          │                                                              │
│          ▼                                                              │
│   ┌──────────────┐     Empty directory created                          │
│   │  emptyDir    │◀─── on node's local disk                            │
│   │   Created    │     (or in memory if specified)                      │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          ▼                                                              │
│   ┌──────────────┐     Containers read/write                            │
│   │  Pod Running │◀─── data to this directory                          │
│   │              │                                                      │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          ▼  Container restarts?                                         │
│   ┌──────────────┐                                                      │
│   │ Data KEPT!   │◀─── emptyDir survives container crashes             │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          ▼  Pod deleted or evicted?                                     │
│   ┌──────────────┐                                                      │
│   │ Data LOST!   │◀─── emptyDir is deleted with Pod                    │
│   └──────────────┘                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 emptyDir Use Cases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      emptyDir USE CASES                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. SCRATCH SPACE FOR COMPUTATION                                       │
│     ┌──────────────────┐                                                │
│     │   Data Process   │                                                │
│     │   ┌───────────┐  │     Large temp files during                    │
│     │   │ /scratch  │◀─┼──── data processing                            │
│     │   └───────────┘  │                                                │
│     └──────────────────┘                                                │
│                                                                         │
│  2. SHARING DATA BETWEEN CONTAINERS                                     │
│     ┌─────────────────────────────────────────────────────────┐         │
│     │                        POD                               │         │
│     │  ┌─────────────┐       ┌─────────────┐                  │         │
│     │  │  Producer   │       │  Consumer   │                  │         │
│     │  │  writes to  │──────▶│ reads from  │                  │         │
│     │  │  /shared    │       │  /shared    │                  │         │
│     │  └─────────────┘       └─────────────┘                  │         │
│     │         │                     │                         │         │
│     │         └─────────┬───────────┘                         │         │
│     │                   ▼                                     │         │
│     │           ┌───────────────┐                             │         │
│     │           │   emptyDir    │                             │         │
│     │           └───────────────┘                             │         │
│     └─────────────────────────────────────────────────────────┘         │
│                                                                         │
│  3. CACHING LAYER                                                       │
│     ┌──────────────────┐                                                │
│     │   Web Server     │                                                │
│     │   ┌───────────┐  │     Cache compiled templates,                  │
│     │   │  /cache   │◀─┼──── resized images, etc.                       │
│     │   └───────────┘  │                                                │
│     └──────────────────┘                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 emptyDir YAML Examples

#### Basic emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from writer" > /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ['sh', '-c', 'sleep 5 && cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

#### emptyDir with Memory Medium (tmpfs)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-memory
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      medium: Memory      # Uses RAM instead of disk
      sizeLimit: 100Mi    # Limit memory usage
```

#### emptyDir with Size Limit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-limited
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: temp-storage
      mountPath: /tmp/data
  volumes:
  - name: temp-storage
    emptyDir:
      sizeLimit: 500Mi    # Pod evicted if exceeded
```

### 3.4 emptyDir Options Reference

| Field | Description | Default |
|-------|-------------|---------|
| `medium` | `""` (disk) or `Memory` (RAM) | `""` |
| `sizeLimit` | Maximum size of the volume | Node's disk/memory |

### 3.5 emptyDir Commands

```bash
# Create the pod
kubectl apply -f emptydir-demo.yaml

# Verify volume is mounted
kubectl exec emptydir-demo -c writer -- ls -la /data

# Check shared data from reader
kubectl exec emptydir-demo -c reader -- cat /data/message.txt

# See where emptyDir is stored on node
kubectl get pod emptydir-demo -o yaml | grep -A2 "volumes:"
```

---

## 4. hostPath Volume

### 4.1 What is hostPath?

A **hostPath** volume mounts a file or directory from the host node's filesystem into your Pod. This is useful for accessing node-level resources.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         hostPath ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │                     KUBERNETES NODE                             │    │
│   │  ┌──────────────────────────────────────────────────────────┐  │    │
│   │  │                   HOST FILESYSTEM                         │  │    │
│   │  │                                                           │  │    │
│   │  │  /var/log/          /data/            /etc/kubernetes/   │  │    │
│   │  │      │                │                     │             │  │    │
│   │  └──────┼────────────────┼─────────────────────┼─────────────┘  │    │
│   │         │                │                     │                │    │
│   │         ▼                ▼                     ▼                │    │
│   │  ┌─────────────────────────────────────────────────────────────┐│    │
│   │  │                       POD                                    ││    │
│   │  │  ┌──────────────────────────────────────────────────────┐   ││    │
│   │  │  │                  CONTAINER                            │   ││    │
│   │  │  │                                                       │   ││    │
│   │  │  │  /host-logs    /host-data    /host-config            │   ││    │
│   │  │  │  (mounted)     (mounted)     (mounted)               │   ││    │
│   │  │  │                                                       │   ││    │
│   │  │  └───────────────────────────────────────────────────────┘   ││    │
│   │  └─────────────────────────────────────────────────────────────┘│    │
│   │                                                                  │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│   ⚠️  WARNING: Pod gets access to node's filesystem!                   │
│   ⚠️  Security risk - use with caution                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 hostPath Types

| Type | Behavior | When to Use |
|------|----------|-------------|
| `""` (empty) | No checks, creates if doesn't exist | Development only |
| `DirectoryOrCreate` | Creates directory if doesn't exist | Safe for new dirs |
| `Directory` | Directory must exist | When path must exist |
| `FileOrCreate` | Creates file if doesn't exist | Safe for new files |
| `File` | File must exist | When file must exist |
| `Socket` | Unix socket must exist | Docker socket access |
| `CharDevice` | Character device must exist | Device access |
| `BlockDevice` | Block device must exist | Raw disk access |

### 4.3 hostPath YAML Examples

#### Basic hostPath

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-data
      mountPath: /data
  volumes:
  - name: host-data
    hostPath:
      path: /data/my-app
      type: DirectoryOrCreate
```

#### Accessing Node Logs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-reader
spec:
  containers:
  - name: log-viewer
    image: busybox
    command: ['sh', '-c', 'tail -f /host-logs/syslog']
    volumeMounts:
    - name: logs
      mountPath: /host-logs
      readOnly: true      # Safety: read-only access
  volumes:
  - name: logs
    hostPath:
      path: /var/log
      type: Directory
```

#### Docker Socket Access (for CI/CD agents)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: docker-agent
spec:
  containers:
  - name: docker
    image: docker:cli
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
```

### 4.4 hostPath Risks and Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      hostPath SECURITY CONCERNS                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ⚠️  RISK 1: Node Escape                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Pod with hostPath: /  can access ENTIRE node filesystem        │   │
│   │  → Container escape → Full node access                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ⚠️  RISK 2: Pod Portability                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  If Pod moves to another node:                                  │   │
│   │  → Different node may not have the same directory               │   │
│   │  → Data is NOT portable between nodes                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ⚠️  RISK 3: Privilege Escalation                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Writing to /etc, /usr, /var can:                               │   │
│   │  → Modify system files                                          │   │
│   │  → Create cron jobs                                             │   │
│   │  → Escalate privileges                                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ✅ BEST PRACTICES:                                                    │
│   • Use readOnly: true whenever possible                                │
│   • Limit to specific subdirectories                                    │
│   • Use type: Directory/File to validate existence                      │
│   • Consider PodSecurityPolicy/Standards to restrict                    │
│   • Prefer PersistentVolumes for production                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Persistent Volumes (PV) and Persistent Volume Claims (PVC)

### 5.1 The Abstraction Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PV/PVC ABSTRACTION MODEL                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   TRADITIONAL APPROACH (Without PV/PVC):                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │   Developer                          Storage Admin               │   │
│   │       │                                    │                     │   │
│   │       │  "I need storage"                  │                     │   │
│   │       ├───────────────────────────────────▶│                     │   │
│   │       │                                    │  Creates storage    │   │
│   │       │  "Here's the NFS path..."          │                     │   │
│   │       │◀───────────────────────────────────┤                     │   │
│   │       │                                    │                     │   │
│   │   Pod YAML has hardcoded NFS details       │                     │   │
│   │   (IP address, path, etc.)                 │                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   KUBERNETES APPROACH (With PV/PVC):                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Developer             Kubernetes          Storage Admin       │   │
│   │       │                     │                    │              │   │
│   │       │                     │   Creates PV       │              │   │
│   │       │                     │◀───────────────────┤              │   │
│   │       │                     │   (Defines actual  │              │   │
│   │       │   Creates PVC       │    storage)        │              │   │
│   │       ├────────────────────▶│                    │              │   │
│   │       │   (Requests storage │                    │              │   │
│   │       │    by size/class)   │                    │              │   │
│   │       │                     │                    │              │   │
│   │       │◀────────────────────┤ Binds PVC to       │              │   │
│   │       │   PVC Bound!        │ matching PV        │              │   │
│   │       │                     │                    │              │   │
│   │   Pod references PVC only (abstracted!)         │              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Benefits:                                                             │
│   • Separation of concerns                                              │
│   • Developer doesn't need storage details                              │
│   • Portable across environments                                        │
│   • Storage can be provisioned in advance or dynamically                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 PV Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PERSISTENT VOLUME LIFECYCLE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐                                                      │
│   │  AVAILABLE   │  PV exists, not bound to any PVC                    │
│   │              │  Ready to be claimed                                 │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          │  PVC requests matching storage                               │
│          ▼                                                              │
│   ┌──────────────┐                                                      │
│   │    BOUND     │  PV is bound to a PVC                               │
│   │              │  One-to-one relationship                             │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          │  PVC is deleted                                              │
│          ▼                                                              │
│   ┌──────────────┐                                                      │
│   │   RELEASED   │  PVC deleted, PV still has data                     │
│   │              │  Not available for new claims                        │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          │  Depends on reclaimPolicy                                    │
│          ├──────────────────┬────────────────────┐                      │
│          ▼                  ▼                    ▼                      │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐               │
│   │   RETAIN     │   │   DELETE     │   │   RECYCLE    │               │
│   │              │   │              │   │  (deprecated) │               │
│   │ Keep data,   │   │ Delete data  │   │ rm -rf, then  │               │
│   │ manual       │   │ and storage  │   │ Available     │               │
│   │ cleanup      │   │ resource     │   │               │               │
│   └──────────────┘   └──────────────┘   └──────────────┘               │
│                                                                         │
│   ┌──────────────┐                                                      │
│   │    FAILED    │  Reclamation failed                                 │
│   │              │  Manual intervention needed                          │
│   └──────────────┘                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Access Modes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ACCESS MODES EXPLAINED                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │             ReadWriteOnce (RWO)                                 │   │
│   │                                                                 │   │
│   │   ┌────────┐                                                    │   │
│   │   │ Node 1 │◀──── Only ONE node can mount read-write            │   │
│   │   │  R/W   │                                                    │   │
│   │   └───┬────┘      ┌────────┐   ┌────────┐                      │   │
│   │       │           │ Node 2 │   │ Node 3 │                      │   │
│   │       ▼           │   ✗    │   │   ✗    │  Cannot mount        │   │
│   │   ┌───────┐       └────────┘   └────────┘                      │   │
│   │   │  PV   │                                                    │   │
│   │   └───────┘       Use: Block storage (EBS, Azure Disk)         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │             ReadOnlyMany (ROX)                                  │   │
│   │                                                                 │   │
│   │   ┌────────┐      ┌────────┐      ┌────────┐                   │   │
│   │   │ Node 1 │      │ Node 2 │      │ Node 3 │                   │   │
│   │   │   RO   │      │   RO   │      │   RO   │ All read-only     │   │
│   │   └───┬────┘      └───┬────┘      └───┬────┘                   │   │
│   │       │               │               │                        │   │
│   │       └───────────────┼───────────────┘                        │   │
│   │                       ▼                                        │   │
│   │                   ┌───────┐                                    │   │
│   │                   │  PV   │  Use: Static content, configs      │   │
│   │                   └───────┘                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │             ReadWriteMany (RWX)                                 │   │
│   │                                                                 │   │
│   │   ┌────────┐      ┌────────┐      ┌────────┐                   │   │
│   │   │ Node 1 │      │ Node 2 │      │ Node 3 │                   │   │
│   │   │  R/W   │      │  R/W   │      │  R/W   │ All read-write    │   │
│   │   └───┬────┘      └───┬────┘      └───┬────┘                   │   │
│   │       │               │               │                        │   │
│   │       └───────────────┼───────────────┘                        │   │
│   │                       ▼                                        │   │
│   │                   ┌───────┐                                    │   │
│   │                   │  PV   │  Use: NFS, CephFS, shared data     │   │
│   │                   └───────┘                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │             ReadWriteOncePod (RWOP) - K8s 1.22+                 │   │
│   │                                                                 │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │ Node 1                                                   │   │   │
│   │   │  ┌───────┐      ┌───────┐                               │   │   │
│   │   │  │ Pod A │      │ Pod B │                               │   │   │
│   │   │  │  R/W  │      │   ✗   │  Only ONE pod, even on        │   │   │
│   │   │  └───┬───┘      └───────┘  same node!                   │   │   │
│   │   │      │                                                   │   │   │
│   │   │      ▼                     Use: Single-writer databases  │   │   │
│   │   │  ┌───────┐                                               │   │   │
│   │   │  │  PV   │                                               │   │   │
│   │   │  └───────┘                                               │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Access Modes Summary Table

| Mode | Short | Meaning | Example Storage |
|------|-------|---------|-----------------|
| ReadWriteOnce | RWO | One node, read-write | AWS EBS, Azure Disk, GCE PD |
| ReadOnlyMany | ROX | Many nodes, read-only | NFS, GlusterFS, CephFS |
| ReadWriteMany | RWX | Many nodes, read-write | NFS, CephFS, Azure Files |
| ReadWriteOncePod | RWOP | One pod only | CSI drivers |

### 5.4 PersistentVolume YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
    environment: production
spec:
  # Storage capacity
  capacity:
    storage: 10Gi
  
  # Access modes (can list multiple)
  accessModes:
    - ReadWriteOnce
  
  # What happens when PVC is deleted
  persistentVolumeReclaimPolicy: Retain
  
  # StorageClass (empty = default)
  storageClassName: standard
  
  # Volume mode (Filesystem or Block)
  volumeMode: Filesystem
  
  # Actual storage backend
  hostPath:
    path: /data/pv-001
    type: DirectoryOrCreate
```

### 5.5 PersistentVolumeClaim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  # What access is needed
  accessModes:
    - ReadWriteOnce
  
  # How much storage
  resources:
    requests:
      storage: 5Gi
  
  # Which StorageClass
  storageClassName: standard
  
  # Volume mode
  volumeMode: Filesystem
  
  # Optional: select specific PV
  selector:
    matchLabels:
      type: local
```

### 5.6 Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc    # Reference the PVC
```

### 5.7 PV/PVC Binding Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      PV/PVC BINDING ALGORITHM                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   PVC Created                                                           │
│       │                                                                 │
│       ▼                                                                 │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │               KUBERNETES CONTROL PLANE                         │     │
│   │                                                                │     │
│   │   1. Find PVs with matching:                                   │     │
│   │      • StorageClass name                                       │     │
│   │      • Access mode(s)                                          │     │
│   │      • Capacity >= requested                                   │     │
│   │      • Selector labels (if specified)                          │     │
│   │                                                                │     │
│   │   2. From matching PVs, select:                                │     │
│   │      • SMALLEST PV that satisfies request                      │     │
│   │        (to minimize wasted space)                              │     │
│   │                                                                │     │
│   │   3. If no match found:                                        │     │
│   │      • Static provisioning: PVC stays Pending                  │     │
│   │      • Dynamic provisioning: Create new PV                     │     │
│   │                                                                │     │
│   │   4. Bind PVC to PV:                                           │     │
│   │      • Set PVC status to Bound                                 │     │
│   │      • Set PV claimRef to PVC                                  │     │
│   │      • Mark PV as Bound                                        │     │
│   │                                                                │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│   Example:                                                              │
│   ┌───────────────────┐         ┌───────────────────┐                  │
│   │       PVC         │         │        PV         │                  │
│   ├───────────────────┤         ├───────────────────┤                  │
│   │ storage: 5Gi      │         │ capacity: 10Gi    │                  │
│   │ accessMode: RWO   │─────────│ accessMode: RWO   │                  │
│   │ class: standard   │         │ class: standard   │                  │
│   │ status: Pending   │         │ status: Available │                  │
│   └───────────────────┘         └───────────────────┘                  │
│              │                           │                              │
│              └───────────┬───────────────┘                              │
│                          ▼                                              │
│   ┌───────────────────┐         ┌───────────────────┐                  │
│   │       PVC         │         │        PV         │                  │
│   ├───────────────────┤         ├───────────────────┤                  │
│   │ storage: 5Gi      │◀────────│ capacity: 10Gi    │                  │
│   │ volumeName: my-pv │         │ claimRef: my-pvc  │                  │
│   │ status: Bound     │         │ status: Bound     │                  │
│   └───────────────────┘         └───────────────────┘                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. StorageClasses and Dynamic Provisioning

### 6.1 What is a StorageClass?

A **StorageClass** provides a way to describe the "classes" of storage available in a cluster. Different classes might map to quality-of-service levels, backup policies, or arbitrary policies determined by the cluster administrators.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STATIC vs DYNAMIC PROVISIONING                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   STATIC PROVISIONING (Manual):                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Admin               Admin               Developer             │   │
│   │     │                   │                     │                 │   │
│   │     │  Create storage   │                     │                 │   │
│   │     ├──────────────────▶│                     │                 │   │
│   │     │  (AWS, NFS, etc)  │                     │                 │   │
│   │     │                   │                     │                 │   │
│   │     │  Create PV        │                     │                 │   │
│   │     ├──────────────────▶│                     │                 │   │
│   │     │                   │                     │                 │   │
│   │     │                   │   Create PVC        │                 │   │
│   │     │                   │◀────────────────────┤                 │   │
│   │     │                   │                     │                 │   │
│   │     │                   │   Bind PVC to PV    │                 │   │
│   │     │                   ├────────────────────▶│                 │   │
│   │                                                                 │   │
│   │   Problem: Manual work, doesn't scale                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   DYNAMIC PROVISIONING (Automatic):                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Admin                    Kubernetes           Cloud/Storage   │   │
│   │     │                          │                     │          │   │
│   │     │  Create StorageClass     │                     │          │   │
│   │     ├─────────────────────────▶│                     │          │   │
│   │     │  (once, defines how      │                     │          │   │
│   │     │   to provision)          │                     │          │   │
│   │     │                          │                     │          │   │
│   │                                │                     │          │   │
│   │   Developer                    │                     │          │   │
│   │     │                          │                     │          │   │
│   │     │  Create PVC with         │                     │          │   │
│   │     │  storageClassName        │                     │          │   │
│   │     ├─────────────────────────▶│                     │          │   │
│   │     │                          │                     │          │   │
│   │     │                          │  Provision storage  │          │   │
│   │     │                          ├────────────────────▶│          │   │
│   │     │                          │                     │          │   │
│   │     │                          │  Create PV          │          │   │
│   │     │                          ├────────────────────▶│          │   │
│   │     │                          │                     │          │   │
│   │     │◀─────────────────────────┤   Bind PVC to PV    │          │   │
│   │     │   PVC Bound!             │                     │          │   │
│   │                                                                 │   │
│   │   Benefit: Fully automated, on-demand provisioning              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Default class
provisioner: kubernetes.io/aws-ebs    # Or any CSI driver
parameters:
  type: gp3                           # Provider-specific
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete                 # or Retain
allowVolumeExpansion: true            # Allow resizing
volumeBindingMode: WaitForFirstConsumer  # or Immediate
mountOptions:
  - hard
  - nfsvers=4.1
```

### 6.3 Common Provisioners

| Cloud/System | Provisioner | Storage Type |
|--------------|-------------|--------------|
| AWS | `kubernetes.io/aws-ebs` | EBS volumes |
| AWS (CSI) | `ebs.csi.aws.com` | EBS via CSI |
| GCP | `kubernetes.io/gce-pd` | Persistent Disk |
| GCP (CSI) | `pd.csi.storage.gke.io` | PD via CSI |
| Azure | `kubernetes.io/azure-disk` | Azure Disk |
| Azure (CSI) | `disk.csi.azure.com` | Disk via CSI |
| Azure Files | `kubernetes.io/azure-file` | SMB shares |
| NFS | Various CSI drivers | NFS shares |
| Local | `kubernetes.io/no-provisioner` | Local disks |
| Rancher | `rancher.io/local-path` | Local path |

### 6.4 Volume Binding Modes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     VOLUME BINDING MODES                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   IMMEDIATE (Default):                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   PVC Created                                                   │   │
│   │       │                                                         │   │
│   │       ▼                                                         │   │
│   │   ┌───────────────┐                                             │   │
│   │   │ PV Provisioned│ ◀── Immediately, in ANY zone               │   │
│   │   │ (e.g., us-east│                                             │   │
│   │   │      -1a)     │                                             │   │
│   │   └───────────────┘                                             │   │
│   │                                                                 │   │
│   │   Pod Scheduled to us-east-1b   ◀── PROBLEM!                   │   │
│   │   Cannot access PV in us-east-1a                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   WAITFORFIRSTCONSUMER (Recommended):                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   PVC Created                                                   │   │
│   │       │                                                         │   │
│   │       ▼                                                         │   │
│   │   ┌───────────────┐                                             │   │
│   │   │ PVC Pending   │ ◀── Waits for Pod to be scheduled          │   │
│   │   │               │                                             │   │
│   │   └───────┬───────┘                                             │   │
│   │           │                                                     │   │
│   │           │  Pod scheduled to us-east-1b                        │   │
│   │           ▼                                                     │   │
│   │   ┌───────────────┐                                             │   │
│   │   │ PV Provisioned│ ◀── In SAME zone as Pod (us-east-1b)       │   │
│   │   │ (us-east-1b)  │                                             │   │
│   │   └───────────────┘                                             │   │
│   │                                                                 │   │
│   │   Pod can access PV!  ◀── SUCCESS!                             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.5 StorageClass Examples by Environment

#### AWS EBS StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### GCP Persistent Disk StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### Local Path StorageClass (for Development)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

---

## 7. ConfigMaps as Volumes

### 7.1 ConfigMap Volume Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGMAP AS VOLUME                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                       ConfigMap                                 │   │
│   │  ┌─────────────────────────────────────────────────────────┐   │   │
│   │  │  data:                                                   │   │   │
│   │  │    config.json: |                                        │   │   │
│   │  │      {"key": "value"}                                    │   │   │
│   │  │    settings.yaml: |                                      │   │   │
│   │  │      debug: true                                         │   │   │
│   │  │    app.properties: |                                     │   │   │
│   │  │      db.host=localhost                                   │   │   │
│   │  └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    │  Mounted as volume                 │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         POD                                     │   │
│   │  ┌─────────────────────────────────────────────────────────┐   │   │
│   │  │                    CONTAINER                             │   │   │
│   │  │                                                          │   │   │
│   │  │   /etc/config/                                           │   │   │
│   │  │   ├── config.json     (file with content from CM)       │   │   │
│   │  │   ├── settings.yaml   (file with content from CM)       │   │   │
│   │  │   └── app.properties  (file with content from CM)       │   │   │
│   │  │                                                          │   │   │
│   │  └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Key Points:                                                           │
│   • Each key becomes a file                                             │
│   • Value becomes file content                                          │
│   • Files are updated when ConfigMap changes (with delay)               │
│   • Mount path should be empty directory                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 ConfigMap Volume Examples

#### Mount Entire ConfigMap

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.json: |
    {
      "database": {
        "host": "db.example.com",
        "port": 5432
      }
    }
  logging.conf: |
    level=INFO
    format=json
---
# Pod
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

#### Mount Specific Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: specific-config-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: config.json
        path: application-config.json    # Rename the file
      - key: logging.conf
        path: log-settings.conf
```

#### Mount as SubPath (Single File Without Overwriting)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d/custom.conf
      subPath: custom.conf    # Only this file, doesn't overwrite directory
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

#### Set File Permissions

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    defaultMode: 0644    # Default permissions
    items:
    - key: script.sh
      path: script.sh
      mode: 0755         # Executable
```

---

## 8. Secrets as Volumes

### 8.1 Secret Volume Overview

Secrets work similarly to ConfigMaps but are designed for sensitive data:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SECRET AS VOLUME                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         Secret                                  │   │
│   │  ┌─────────────────────────────────────────────────────────┐   │   │
│   │  │  type: Opaque                                            │   │   │
│   │  │  data:                                                   │   │   │
│   │  │    username: YWRtaW4=        (base64 encoded)           │   │   │
│   │  │    password: cGFzc3dvcmQ=    (base64 encoded)           │   │   │
│   │  │    tls.crt: LS0tLS1C...      (certificate)              │   │   │
│   │  │    tls.key: LS0tLS1C...      (private key)              │   │   │
│   │  └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    │  Mounted as volume                 │
│                                    │  (automatically decoded)           │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         POD                                     │   │
│   │  ┌─────────────────────────────────────────────────────────┐   │   │
│   │  │                    CONTAINER                             │   │   │
│   │  │                                                          │   │   │
│   │  │   /etc/secrets/                                          │   │   │
│   │  │   ├── username    (contains: admin)                     │   │   │
│   │  │   ├── password    (contains: password)                  │   │   │
│   │  │   ├── tls.crt     (contains: certificate)               │   │   │
│   │  │   └── tls.key     (contains: private key)               │   │   │
│   │  │                                                          │   │   │
│   │  │   Mode: 0644 by default (or custom)                     │   │   │
│   │  │   tmpfs: Stored in memory, not on disk!                 │   │   │
│   │  │                                                          │   │   │
│   │  └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Security Notes:                                                       │
│   • Secrets are stored in tmpfs (RAM), not disk                         │
│   • Base64 decoded when mounted                                         │
│   • Default mode 0644 (consider 0400 for sensitive)                     │
│   • Enable encryption at rest in etcd for production                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Secret Volume Examples

#### Basic Secret Mount

```yaml
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=      # postgres
  password: c2VjcmV0MTIz      # secret123
---
# Pod
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400    # Restrict permissions
```

#### TLS Certificates

```yaml
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...
  tls.key: LS0tLS1CRUdJTi...
---
# Pod
apiVersion: v1
kind: Pod
metadata:
  name: https-server
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 443
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/nginx/ssl
      readOnly: true
  volumes:
  - name: tls-certs
    secret:
      secretName: tls-secret
```

---

## 9. Projected Volumes

### 9.1 What are Projected Volumes?

A **projected** volume maps several volume sources into a single directory:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       PROJECTED VOLUME                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐              │
│   │   ConfigMap   │  │    Secret     │  │  DownwardAPI  │              │
│   │               │  │               │  │               │              │
│   │  app.conf     │  │  password     │  │  pod-name     │              │
│   │  logging.conf │  │  api-key      │  │  namespace    │              │
│   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘              │
│           │                  │                  │                       │
│           └──────────────────┼──────────────────┘                       │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    PROJECTED VOLUME                             │   │
│   │                    /etc/app-data/                               │   │
│   │                                                                 │   │
│   │    ├── app.conf        (from ConfigMap)                        │   │
│   │    ├── logging.conf    (from ConfigMap)                        │   │
│   │    ├── password        (from Secret)                           │   │
│   │    ├── api-key         (from Secret)                           │   │
│   │    ├── pod-name        (from DownwardAPI)                      │   │
│   │    └── namespace       (from DownwardAPI)                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Benefits:                                                             │
│   • Combine multiple sources in one mount                               │
│   • Single mount point for related config                               │
│   • Cleaner pod spec                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Projected Volume YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: all-config
      mountPath: /etc/app-config
  volumes:
  - name: all-config
    projected:
      sources:
      # ConfigMap source
      - configMap:
          name: app-config
          items:
          - key: config.json
            path: config.json
      # Secret source
      - secret:
          name: app-secrets
          items:
          - key: api-key
            path: credentials/api-key
      # DownwardAPI source
      - downwardAPI:
          items:
          - path: pod-info/name
            fieldRef:
              fieldPath: metadata.name
          - path: pod-info/namespace
            fieldRef:
              fieldPath: metadata.namespace
      # ServiceAccountToken source
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: vault
```

---

## 10. Volume Expansion

### 10.1 Expanding PVCs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      VOLUME EXPANSION WORKFLOW                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Prerequisites:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  StorageClass must have:                                        │   │
│   │    allowVolumeExpansion: true                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Expansion Process:                                                    │
│                                                                         │
│   1. Edit PVC to request more storage:                                  │
│      ┌────────────────────────────────────────┐                        │
│      │  kubectl edit pvc my-pvc               │                        │
│      │  # Change: storage: 5Gi → 10Gi         │                        │
│      └────────────────────────────────────────┘                        │
│                          │                                              │
│                          ▼                                              │
│   2. Controller expands underlying storage:                             │
│      ┌────────────────────────────────────────┐                        │
│      │  PV: 5Gi → 10Gi (cloud provider API)   │                        │
│      └────────────────────────────────────────┘                        │
│                          │                                              │
│                          ▼                                              │
│   3. Filesystem resize (may need pod restart):                          │
│      ┌────────────────────────────────────────┐                        │
│      │  If filesystem resize needed:          │                        │
│      │  - Delete and recreate pod, OR         │                        │
│      │  - Online resize (if supported)        │                        │
│      └────────────────────────────────────────┘                        │
│                          │                                              │
│                          ▼                                              │
│   4. PVC shows new size:                                                │
│      ┌────────────────────────────────────────┐                        │
│      │  kubectl get pvc my-pvc                │                        │
│      │  CAPACITY: 10Gi ✓                      │                        │
│      └────────────────────────────────────────┘                        │
│                                                                         │
│   ⚠️  Note: You can only INCREASE size, never decrease!               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Expansion Commands

```bash
# Check if StorageClass supports expansion
kubectl get sc -o jsonpath='{.items[*].allowVolumeExpansion}'

# Expand PVC
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# Or edit directly
kubectl edit pvc my-pvc

# Check expansion status
kubectl get pvc my-pvc -o yaml | grep -A5 conditions
```

---

## 11. Practical Examples

### 11.1 Complete Database Deployment with Persistent Storage

```yaml
# StorageClass for database
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# PVC for PostgreSQL
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: database-storage
  resources:
    requests:
      storage: 20Gi
---
# PostgreSQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

### 11.2 Application with ConfigMap and Secret

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      pool_size: 10
      timeout: 30s
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: supersecret123
  api-key: sk_live_xxxxxxxxxxxx
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: secrets
          mountPath: /app/secrets
          readOnly: true
        - name: cache
          mountPath: /app/cache
      volumes:
      - name: config
        configMap:
          name: app-config
      - name: secrets
        secret:
          secretName: app-secrets
          defaultMode: 0400
      - name: cache
        emptyDir:
          sizeLimit: 500Mi
```

---

## 12. Volume Commands Cheatsheet

### PersistentVolume Commands

```bash
# List all PVs
kubectl get pv

# Get PV details
kubectl describe pv <pv-name>

# Get PVs with more info
kubectl get pv -o wide

# Check PV status
kubectl get pv <pv-name> -o jsonpath='{.status.phase}'

# Delete PV (must be Released first)
kubectl delete pv <pv-name>
```

### PersistentVolumeClaim Commands

```bash
# List all PVCs
kubectl get pvc

# Get PVC in namespace
kubectl get pvc -n <namespace>

# Get PVC details
kubectl describe pvc <pvc-name>

# Check bound volume
kubectl get pvc <pvc-name> -o jsonpath='{.spec.volumeName}'

# Expand PVC
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Delete PVC
kubectl delete pvc <pvc-name>
```

### StorageClass Commands

```bash
# List StorageClasses
kubectl get sc

# Get default StorageClass
kubectl get sc -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'

# Describe StorageClass
kubectl describe sc <sc-name>

# Create StorageClass from file
kubectl apply -f storageclass.yaml
```

### Volume Troubleshooting

```bash
# Check why PVC is pending
kubectl describe pvc <pvc-name> | grep -A5 Events

# Check volume in pod
kubectl exec <pod> -- df -h

# Check mounted volumes
kubectl exec <pod> -- mount | grep <path>

# Verify ConfigMap/Secret mount
kubectl exec <pod> -- ls -la /path/to/mount

# Check volume content
kubectl exec <pod> -- cat /path/to/mount/file
```

---

## 13. CKA Exam Tips

### High-Priority Topics

| Topic | Exam Weight | Key Skills |
|-------|-------------|------------|
| Create PV/PVC | 🔴 HIGH | Write YAML from scratch |
| Mount volumes in Pods | 🔴 HIGH | volumeMounts, volumes |
| StorageClass | 🔴 HIGH | Dynamic provisioning |
| emptyDir | 🟡 MEDIUM | Sharing between containers |
| hostPath | 🟡 MEDIUM | Node-level access |
| ConfigMap volumes | 🔴 HIGH | Mount as files |
| Secret volumes | 🔴 HIGH | Mount as files |

### Quick Reference for Exam

```yaml
# Minimal PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data

# Minimal PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

# Use in Pod
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

### Common Exam Mistakes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA MISTAKES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ❌ Mistake 1: PVC requests more than PV capacity                      │
│      PV: 5Gi, PVC: 10Gi → Pending forever!                             │
│                                                                         │
│   ❌ Mistake 2: Access mode mismatch                                    │
│      PV: ReadWriteOnce, PVC: ReadWriteMany → No binding                │
│                                                                         │
│   ❌ Mistake 3: StorageClass mismatch                                   │
│      PV: storageClassName: fast, PVC: standard → No binding            │
│                                                                         │
│   ❌ Mistake 4: Forgetting volumeMounts in container                    │
│      Volume defined but not mounted → Data not accessible              │
│                                                                         │
│   ❌ Mistake 5: Mount path overwrites important directory              │
│      mountPath: /etc → Overwrites container's /etc!                    │
│      Use subPath for single files                                       │
│                                                                         │
│   ❌ Mistake 6: Wrong hostPath type                                     │
│      type: Directory but path doesn't exist → Pod fails                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Docker to Kubernetes Volume Mapping

| Docker Command | Kubernetes Equivalent |
|---------------|----------------------|
| `docker run -v mydata:/app/data` | PVC mounted in Pod |
| `docker run -v /host:/container` | hostPath volume |
| `docker run --tmpfs /tmp` | emptyDir with `medium: Memory` |
| `docker volume create` | PV creation |
| `docker run -v $(pwd)/config.json:/app/config.json` | ConfigMap mounted as file |
| `docker run --env-file .env` | ConfigMap/Secret as env or volume |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VOLUME TYPES DECISION TREE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Need Storage?                                                         │
│       │                                                                 │
│       ├──▶ Temporary (lost with Pod)?                                  │
│       │       │                                                         │
│       │       ├──▶ In RAM? ─────▶ emptyDir {medium: Memory}            │
│       │       │                                                         │
│       │       └──▶ On disk? ────▶ emptyDir {}                          │
│       │                                                                 │
│       ├──▶ Persist across Pod restarts?                                │
│       │       │                                                         │
│       │       ├──▶ Single node dev? ─▶ hostPath                        │
│       │       │                                                         │
│       │       └──▶ Production? ─────▶ PV + PVC + StorageClass          │
│       │                                                                 │
│       ├──▶ Configuration files?                                        │
│       │       │                                                         │
│       │       ├──▶ Non-sensitive? ──▶ ConfigMap volume                 │
│       │       │                                                         │
│       │       └──▶ Sensitive? ──────▶ Secret volume                    │
│       │                                                                 │
│       └──▶ Multiple sources in one mount? ─▶ projected volume          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **ConfigMaps** - Non-sensitive configuration management
- **Secrets** - Sensitive data management
- Creating, using, and updating ConfigMaps and Secrets
- Best practices for configuration management

---

**Chapter 16 Complete! 🎉**

You now understand:
- Volume types and when to use each
- emptyDir for temporary storage
- hostPath for node-level access
- PV/PVC for persistent storage
- StorageClasses for dynamic provisioning
- ConfigMaps and Secrets as volumes
- Projected volumes for combining sources
- Volume expansion
- CKA exam preparation for volumes

