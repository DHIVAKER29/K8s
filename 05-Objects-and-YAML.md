# ☸️ Chapter 05: Kubernetes Objects & YAML

> Understanding Kubernetes API objects and mastering YAML manifests - the foundation of everything in Kubernetes.

---

## 📚 Table of Contents

1. [What are Kubernetes Objects?](#-what-are-kubernetes-objects)
2. [Object Categories](#-object-categories)
3. [YAML Manifest Structure](#-yaml-manifest-structure)
4. [Required Fields](#-required-fields)
5. [Labels and Selectors](#-labels-and-selectors)
6. [Annotations](#-annotations)
7. [Namespaces](#-namespaces)
8. [Common Object Types](#-common-object-types)
9. [Multi-Resource Manifests](#-multi-resource-manifests)
10. [Object Relationships](#-object-relationships)
11. [Best Practices](#-best-practices)
12. [CKA Exam Tips](#-cka-exam-tips)
13. [Summary](#-summary)

---

## 📖 What are Kubernetes Objects?

### Definition

> **Kubernetes Objects** are persistent entities in the Kubernetes system that represent the state of your cluster. They describe what containerized applications are running, what resources are available, and the policies around how those applications behave.

### Key Concept: Desired State vs Current State

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES OBJECT MODEL                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          YAML MANIFEST                                      │   │
│  │                                                                             │   │
│  │  apiVersion: apps/v1                                                        │   │
│  │  kind: Deployment                                                           │   │
│  │  metadata:                                                                  │   │
│  │    name: nginx                         ◄─── "I want 3 nginx pods"          │   │
│  │  spec:                                                                      │   │
│  │    replicas: 3                                                              │   │
│  │    ...                                                                      │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                           │
│                           kubectl apply -f                                          │
│                                         │                                           │
│                                         ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         KUBERNETES API                                      │   │
│  │                                                                             │   │
│  │   Stores object in etcd as "desired state"                                 │   │
│  │   Controllers continuously work to achieve this state                      │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                           │
│                                         ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                        CURRENT STATE (REALITY)                              │   │
│  │                                                                             │   │
│  │    ┌─────┐    ┌─────┐    ┌─────┐                                           │   │
│  │    │Pod 1│    │Pod 2│    │Pod 3│   ◄─── 3 pods running = desired met       │   │
│  │    │nginx│    │nginx│    │nginx│                                           │   │
│  │    └─────┘    └─────┘    └─────┘                                           │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  If current ≠ desired, Kubernetes takes action to reconcile                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Object Properties

Every Kubernetes object has:

| Property | Description | Example |
|----------|-------------|---------|
| **apiVersion** | API group and version | `v1`, `apps/v1` |
| **kind** | Type of object | `Pod`, `Deployment` |
| **metadata** | Data that identifies the object | `name`, `namespace`, `labels` |
| **spec** | Desired state | `replicas: 3`, `image: nginx` |
| **status** | Current state (managed by K8s) | `readyReplicas: 3` |

---

## 📂 Object Categories

### Workload Objects (Running Your Apps)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         WORKLOAD OBJECTS                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Object          │ Purpose                         │ Use Case                       │
│  ────────────────┼─────────────────────────────────┼────────────────────────────────│
│  Pod             │ Smallest deployable unit        │ Single container or sidecar    │
│  ReplicaSet      │ Maintains pod replicas          │ Usually managed by Deployment  │
│  Deployment      │ Declarative pod management      │ Stateless applications        │
│  StatefulSet     │ Ordered, stable pod management  │ Databases, stateful apps      │
│  DaemonSet       │ Pod on every node               │ Logging, monitoring agents    │
│  Job             │ Run to completion               │ Batch processing             │
│  CronJob         │ Scheduled jobs                  │ Periodic tasks               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Service Objects (Networking)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE & NETWORKING OBJECTS                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Object          │ Purpose                         │ Use Case                       │
│  ────────────────┼─────────────────────────────────┼────────────────────────────────│
│  Service         │ Stable endpoint for pods        │ Load balancing, discovery     │
│  Endpoints       │ IP addresses backing a service  │ Auto-managed by Service       │
│  Ingress         │ HTTP/HTTPS routing              │ External access, SSL          │
│  NetworkPolicy   │ Pod network rules               │ Security, isolation           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Configuration Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         CONFIGURATION OBJECTS                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Object          │ Purpose                         │ Use Case                       │
│  ────────────────┼─────────────────────────────────┼────────────────────────────────│
│  ConfigMap       │ Non-sensitive configuration     │ Config files, env vars        │
│  Secret          │ Sensitive data                  │ Passwords, tokens, certs      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Storage Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         STORAGE OBJECTS                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Object              │ Purpose                      │ Use Case                      │
│  ────────────────────┼──────────────────────────────┼───────────────────────────────│
│  PersistentVolume    │ Cluster storage resource     │ Provisioned storage           │
│  PersistentVolumeClaim│ Request for storage         │ Pod storage request           │
│  StorageClass        │ Storage provisioner config   │ Dynamic provisioning          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Cluster Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         CLUSTER & SECURITY OBJECTS                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Object              │ Purpose                      │ Scope                         │
│  ────────────────────┼──────────────────────────────┼───────────────────────────────│
│  Namespace           │ Virtual cluster              │ Cluster                       │
│  Node                │ Worker machine               │ Cluster                       │
│  ServiceAccount      │ Pod identity                 │ Namespace                     │
│  Role                │ Namespace permissions        │ Namespace                     │
│  ClusterRole         │ Cluster-wide permissions     │ Cluster                       │
│  RoleBinding         │ Bind role to user/sa         │ Namespace                     │
│  ClusterRoleBinding  │ Bind clusterrole             │ Cluster                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 YAML Manifest Structure

### The Four Required Fields

```yaml
# EVERY Kubernetes YAML needs these 4 fields:

apiVersion: v1           # 1. API version
kind: Pod                # 2. Resource type
metadata:                # 3. Metadata
  name: my-pod
spec:                    # 4. Specification
  containers:
  - name: nginx
    image: nginx
```

### Detailed Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         YAML MANIFEST STRUCTURE                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  apiVersion: apps/v1      ◄─── API group/version for this resource type            │
│  kind: Deployment         ◄─── What type of resource                               │
│  metadata:                ◄─── Identifying information                             │
│    name: nginx-deployment         # Required: unique name                           │
│    namespace: default             # Optional: defaults to 'default'                 │
│    labels:                        # Optional: key-value for selection              │
│      app: nginx                                                                     │
│      tier: frontend                                                                 │
│    annotations:                   # Optional: non-identifying metadata             │
│      description: "My nginx app"                                                    │
│  spec:                    ◄─── Desired state (varies by kind)                      │
│    replicas: 3                                                                      │
│    selector:                                                                        │
│      matchLabels:                                                                   │
│        app: nginx                                                                   │
│    template:                      # Pod template                                    │
│      metadata:                                                                      │
│        labels:                                                                      │
│          app: nginx                                                                 │
│      spec:                        # Pod spec                                        │
│        containers:                                                                  │
│        - name: nginx                                                                │
│          image: nginx:1.19                                                          │
│          ports:                                                                     │
│          - containerPort: 80                                                        │
│  status:                  ◄─── Current state (managed by Kubernetes)               │
│    replicas: 3                    # You don't write this                            │
│    readyReplicas: 3               # K8s populates it                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Required Fields

### 1. apiVersion

```yaml
# Core API (no group prefix)
apiVersion: v1           # Pod, Service, ConfigMap, Secret, Namespace, etc.

# Named API groups
apiVersion: apps/v1              # Deployment, ReplicaSet, StatefulSet, DaemonSet
apiVersion: batch/v1             # Job, CronJob
apiVersion: networking.k8s.io/v1 # Ingress, NetworkPolicy
apiVersion: rbac.authorization.k8s.io/v1  # Role, ClusterRole, RoleBinding
apiVersion: autoscaling/v2       # HorizontalPodAutoscaler
apiVersion: storage.k8s.io/v1    # StorageClass
```

```bash
# Find the correct apiVersion
kubectl api-resources | grep Deployment
# Output: deployments   deploy   apps/v1   true   Deployment

kubectl api-versions | grep apps
# Output: apps/v1
```

### 2. kind

```yaml
# Must match exactly (case-sensitive)
kind: Pod
kind: Deployment
kind: Service
kind: ConfigMap
kind: Secret
kind: PersistentVolumeClaim
kind: Ingress
kind: NetworkPolicy
```

### 3. metadata

```yaml
metadata:
  # Required
  name: my-resource           # Unique within namespace
  
  # Optional
  namespace: production       # Omit for cluster-scoped resources
  labels:                     # For selection and organization
    app: nginx
    environment: production
    team: platform
  annotations:                # Non-identifying metadata
    kubernetes.io/description: "Production nginx deployment"
    prometheus.io/scrape: "true"
  
  # Read-only (set by Kubernetes)
  uid: 12345-abcde-67890     # Unique identifier
  resourceVersion: "12345"   # Version for optimistic locking
  creationTimestamp: "2024-01-15T10:00:00Z"
  generation: 1              # Spec change counter
```

### 4. spec

The `spec` field varies significantly by resource type:

```yaml
# Pod spec
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80

# Deployment spec
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

# Service spec
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080
```

---

## 🏷️ Labels and Selectors

### What are Labels?

> **Labels** are key-value pairs attached to objects that are used to identify and select groups of objects.

```yaml
metadata:
  labels:
    app: nginx                    # Application name
    environment: production       # Environment
    tier: frontend               # Architecture tier
    team: platform               # Owning team
    version: v1.2.3              # Version
    release: stable              # Release channel
```

### Label Rules

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         LABEL NAMING RULES                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Keys:                                                                              │
│  • Optional prefix (DNS subdomain) + "/" + name                                     │
│  • Prefix max 253 chars, name max 63 chars                                         │
│  • Must start/end with alphanumeric                                                │
│  • Can contain: - _ . alphanumeric                                                 │
│                                                                                      │
│  Values:                                                                            │
│  • Max 63 characters (can be empty)                                                │
│  • Must start/end with alphanumeric (if not empty)                                 │
│  • Can contain: - _ . alphanumeric                                                 │
│                                                                                      │
│  Examples:                                                                          │
│  ✅ app: nginx                                                                      │
│  ✅ app.kubernetes.io/name: nginx                                                   │
│  ✅ team: platform-engineering                                                      │
│  ✅ version: v1.2.3                                                                 │
│  ❌ App: Nginx (uppercase not recommended)                                          │
│  ❌ my label: value (no spaces)                                                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Recommended Labels

```yaml
# Kubernetes recommended labels (app.kubernetes.io/*)
metadata:
  labels:
    app.kubernetes.io/name: nginx              # Application name
    app.kubernetes.io/instance: nginx-prod     # Instance identifier
    app.kubernetes.io/version: "1.19.0"        # Version
    app.kubernetes.io/component: webserver     # Component within app
    app.kubernetes.io/part-of: website         # Higher-level app
    app.kubernetes.io/managed-by: helm         # Tool managing this
```

### Selectors

Selectors are used to select objects based on their labels:

```yaml
# Equality-based selectors
selector:
  matchLabels:
    app: nginx
    environment: production

# Set-based selectors
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
```

### Selector Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `In` | Value is in set | `app In [nginx, apache]` |
| `NotIn` | Value not in set | `env NotIn [dev, test]` |
| `Exists` | Key exists | `tier Exists` |
| `DoesNotExist` | Key doesn't exist | `deprecated DoesNotExist` |

### Where Selectors Are Used

```yaml
# Service selecting pods
apiVersion: v1
kind: Service
spec:
  selector:           # Equality-based only
    app: nginx

# Deployment selecting pods
apiVersion: apps/v1
kind: Deployment
spec:
  selector:
    matchLabels:      # Can use matchLabels AND/OR matchExpressions
      app: nginx
    matchExpressions:
    - key: environment
      operator: In
      values: [production, staging]

# NetworkPolicy selecting pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  podSelector:
    matchLabels:
      role: db
```

### Using Labels with kubectl

```bash
# Show labels
kubectl get pods --show-labels

# Show specific labels as columns
kubectl get pods -L app,environment

# Filter by label
kubectl get pods -l app=nginx
kubectl get pods -l 'app=nginx,environment=production'
kubectl get pods -l 'app in (nginx,apache)'
kubectl get pods -l 'environment!=production'
kubectl get pods -l 'tier'                    # Has label
kubectl get pods -l '!deprecated'             # Doesn't have label

# Add label
kubectl label pod nginx environment=production

# Update label
kubectl label pod nginx environment=staging --overwrite

# Remove label
kubectl label pod nginx environment-

# Label multiple resources
kubectl label pods -l app=nginx tier=frontend
```

---

## 📝 Annotations

### What are Annotations?

> **Annotations** are key-value pairs for attaching non-identifying metadata to objects. They're not used for selection but for storing additional information.

```yaml
metadata:
  annotations:
    # Descriptive
    description: "Main production nginx deployment"
    owner: "platform-team@company.com"
    
    # Tool configuration
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    
    # Ingress configuration
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # Deployment info
    kubernetes.io/change-cause: "Update to nginx 1.19"
    
    # Build info
    build.number: "1234"
    git.commit: "abc123def456"
```

### Labels vs Annotations

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         LABELS VS ANNOTATIONS                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Aspect           │ Labels                    │ Annotations                         │
│  ─────────────────┼───────────────────────────┼─────────────────────────────────────│
│  Purpose          │ Identify and select       │ Store metadata                      │
│  Querying         │ Can query/filter          │ Cannot query/filter                 │
│  Value size       │ Max 63 chars              │ Much larger (256KB total)           │
│  Used by          │ Kubernetes core           │ External tools, humans              │
│  Selection        │ Used in selectors         │ Not used in selectors               │
│                                                                                      │
│  Use Labels for:                                                                    │
│  • Grouping related resources                                                       │
│  • Selecting targets (Services, Deployments)                                        │
│  • Filtering with kubectl                                                           │
│                                                                                      │
│  Use Annotations for:                                                               │
│  • Build/release information                                                        │
│  • Tool-specific configuration                                                      │
│  • Contact information                                                              │
│  • Descriptions                                                                     │
│  • Long values (JSON, URLs)                                                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🗂️ Namespaces

### What are Namespaces?

> **Namespaces** provide a way to divide cluster resources between multiple users, teams, or projects. They're like virtual clusters within a physical cluster.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES NAMESPACES                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Physical Kubernetes Cluster                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐   │ │
│  │  │   default           │  │   production        │  │   development       │   │ │
│  │  │   namespace         │  │   namespace         │  │   namespace         │   │ │
│  │  │                     │  │                     │  │                     │   │ │
│  │  │  ┌───┐ ┌───┐       │  │  ┌───┐ ┌───┐ ┌───┐ │  │  ┌───┐ ┌───┐       │   │ │
│  │  │  │Pod│ │Svc│       │  │  │Pod│ │Pod│ │Svc│ │  │  │Pod│ │Svc│       │   │ │
│  │  │  └───┘ └───┘       │  │  └───┘ └───┘ └───┘ │  │  └───┘ └───┘       │   │ │
│  │  │                     │  │                     │  │                     │   │ │
│  │  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘   │ │
│  │                                                                               │ │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                            │ │
│  │  │   kube-system       │  │   kube-public       │  System namespaces         │ │
│  │  │   (K8s components)  │  │   (public data)     │                            │ │
│  │  └─────────────────────┘  └─────────────────────┘                            │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  Resources are isolated within namespaces (names can repeat across namespaces)     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Default Namespaces

| Namespace | Purpose |
|-----------|---------|
| `default` | Default for objects with no namespace |
| `kube-system` | Kubernetes system components |
| `kube-public` | Publicly accessible data |
| `kube-node-lease` | Node heartbeat leases |

### Namespace Operations

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create namespace
kubectl create namespace development
kubectl create ns staging

# Or via YAML
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production

# Delete namespace (deletes ALL resources in it!)
kubectl delete namespace development

# Set default namespace
kubectl config set-context --current --namespace=production

# View current namespace
kubectl config view --minify | grep namespace
```

### Namespaced vs Cluster-Scoped

```bash
# Check if resource is namespaced
kubectl api-resources --namespaced=true   # Namespaced resources
kubectl api-resources --namespaced=false  # Cluster-scoped resources
```

| Namespaced | Cluster-Scoped |
|------------|----------------|
| Pod, Deployment, Service | Node |
| ConfigMap, Secret | PersistentVolume |
| Role, RoleBinding | ClusterRole, ClusterRoleBinding |
| PersistentVolumeClaim | Namespace |
| Ingress, NetworkPolicy | StorageClass |

### Cross-Namespace Communication

```yaml
# Accessing services across namespaces
# Format: <service>.<namespace>.svc.cluster.local

# From any namespace, access postgres in "database" namespace:
postgres.database.svc.cluster.local

# Short form (within cluster):
postgres.database
```

---

## 📦 Common Object Types

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.19
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    env:
    - name: ENV_VAR
      value: "value"
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
  restartPolicy: Always
```

### Deployment

```yaml
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
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
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
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP           # ClusterIP, NodePort, LoadBalancer, ExternalName
  selector:
    app: nginx              # Selects pods with this label
  ports:
  - name: http
    port: 80                # Service port
    targetPort: 80          # Container port
    protocol: TCP
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value
  database_host: "postgres"
  database_port: "5432"
  
  # Multi-line value (file-like)
  config.yaml: |
    server:
      port: 8080
    logging:
      level: info
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque              # Default type
data:
  # Values must be base64 encoded
  username: YWRtaW4=      # echo -n "admin" | base64
  password: cGFzc3dvcmQ=  # echo -n "password" | base64

# Or use stringData (plain text, converted to base64)
stringData:
  api-key: "my-secret-api-key"
```

---

## 📄 Multi-Resource Manifests

### Single File with Multiple Resources

```yaml
# app.yaml - Multiple resources separated by ---

apiVersion: v1
kind: Namespace
metadata:
  name: myapp

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  APP_ENV: production

---

apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
stringData:
  DB_PASSWORD: secretpassword

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
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
      - name: myapp
        image: myapp:v1
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets

---

apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Apply all resources
kubectl apply -f app.yaml

# Delete all resources
kubectl delete -f app.yaml
```

### Directory of Files

```
myapp/
├── 00-namespace.yaml
├── 01-configmap.yaml
├── 02-secret.yaml
├── 03-deployment.yaml
└── 04-service.yaml
```

```bash
# Apply all files in directory
kubectl apply -f myapp/

# Apply recursively
kubectl apply -f myapp/ -R
```

---

## 🔗 Object Relationships

### Ownership (Controller Pattern)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         OBJECT OWNERSHIP HIERARCHY                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Deployment                                                                          │
│      │                                                                               │
│      │ owns (creates and manages)                                                   │
│      ▼                                                                               │
│  ReplicaSet                                                                          │
│      │                                                                               │
│      │ owns (creates and manages)                                                   │
│      ▼                                                                               │
│  Pod ──────────────────────────────────────────────────────────────────────────────│
│                                                                                      │
│  When you delete a Deployment:                                                      │
│  • Its ReplicaSets are automatically deleted                                        │
│  • Which causes its Pods to be automatically deleted                               │
│                                                                                      │
│  This is "cascading delete" - controlled by ownerReferences                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Selection (Service → Pods)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE SELECTION                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Service                     Pods                                                   │
│  ┌─────────────────────┐     ┌────────────────────────┐                            │
│  │ metadata:           │     │ metadata:              │                            │
│  │   name: my-service  │     │   name: pod-1          │                            │
│  │ spec:               │     │   labels:              │                            │
│  │   selector:         │────>│     app: nginx   ◄────│── Matches!                  │
│  │     app: nginx      │     │     tier: frontend     │                            │
│  └─────────────────────┘     └────────────────────────┘                            │
│                              ┌────────────────────────┐                            │
│                              │ metadata:              │                            │
│                              │   name: pod-2          │                            │
│                         ────>│   labels:              │                            │
│                              │     app: nginx   ◄────│── Matches!                  │
│                              └────────────────────────┘                            │
│                              ┌────────────────────────┐                            │
│                              │ metadata:              │                            │
│                              │   name: pod-3          │                            │
│                         ────>│   labels:              │                            │
│                              │     app: apache  ◄────│── Does NOT match           │
│                              └────────────────────────┘                            │
│                                                                                      │
│  Service routes traffic to pods matching its selector                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ✨ Best Practices

### YAML Best Practices

```yaml
# 1. Always specify apiVersion and kind
apiVersion: apps/v1
kind: Deployment

# 2. Use meaningful names
metadata:
  name: payment-service    # Good
  # name: ps              # Bad (unclear)

# 3. Always use labels
metadata:
  labels:
    app: payment-service
    environment: production
    team: payments

# 4. Always specify resource requests/limits
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"

# 5. Use specific image tags
image: nginx:1.19.0        # Good
# image: nginx             # Bad (defaults to :latest)
# image: nginx:latest      # Bad (mutable)

# 6. Add liveness and readiness probes
livenessProbe:
  httpGet:
    path: /health
    port: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

# 7. Use namespaces for organization
metadata:
  namespace: production
```

### Labeling Strategy

```yaml
# Recommended label schema
labels:
  # What
  app.kubernetes.io/name: nginx
  app.kubernetes.io/component: webserver
  app.kubernetes.io/part-of: website
  app.kubernetes.io/version: "1.19.0"
  
  # Who
  app.kubernetes.io/managed-by: helm
  team: platform
  
  # Where
  environment: production
  region: us-east-1
```

---

## 🎓 CKA Exam Tips

### Quick Object Creation

```bash
# Generate YAML templates
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip nginx --tcp=80 --dry-run=client -o yaml > svc.yaml

# Use kubectl explain
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
```

### Common Exam Patterns

```bash
# Create pod with specific labels
kubectl run nginx --image=nginx --labels="app=nginx,tier=frontend"

# Create deployment with specific replicas
kubectl create deployment nginx --image=nginx --replicas=3

# Expose deployment
kubectl expose deployment nginx --port=80 --type=NodePort

# Add labels to existing resources
kubectl label pod nginx environment=production

# Change namespace for subsequent commands
kubectl config set-context --current --namespace=my-namespace
```

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Object** | Persistent entity representing cluster state |
| **Manifest** | YAML file defining desired state |
| **apiVersion** | API group and version |
| **kind** | Type of resource |
| **metadata** | Identifying information (name, labels) |
| **spec** | Desired state |
| **status** | Current state (managed by K8s) |
| **Labels** | Key-value pairs for selection |
| **Annotations** | Non-identifying metadata |
| **Namespace** | Virtual cluster for isolation |

### Object Hierarchy

```
Cluster
├── Nodes (cluster-scoped)
├── Namespaces
│   ├── Deployments
│   │   └── ReplicaSets
│   │       └── Pods
│   ├── Services
│   ├── ConfigMaps
│   ├── Secrets
│   └── ...
└── PersistentVolumes (cluster-scoped)
```

---

## 🔜 What's Next

In **Chapter 06: Pods Deep Dive**, we'll cover:

- Pod anatomy and lifecycle
- Multi-container pods (sidecars, init containers)
- Pod networking
- Resource requests and limits
- Probes (liveness, readiness, startup)
- Pod scheduling

---

*Understanding objects and YAML is fundamental - everything in Kubernetes is an object!*

