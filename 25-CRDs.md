# Chapter 25: Custom Resources (CRDs)

## Introduction

Kubernetes provides built-in resources like Pods, Deployments, Services. But what if you need resources specific to your application? **Custom Resource Definitions (CRDs)** let you extend the Kubernetes API with your own resource types.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   EXTENDING KUBERNETES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Built-in Resources:                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Pod    Deployment    Service    ConfigMap    Secret          │   │
│   │   Job    StatefulSet   Ingress    PV/PVC       ...             │   │
│   │                                                                 │   │
│   │   These are built into Kubernetes                               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Custom Resources (Your Extensions):                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Database    Certificate    Backup    GitRepository           │   │
│   │   Kafka       Redis         MySQL     PostgresCluster          │   │
│   │                                                                 │   │
│   │   You define these with CRDs!                                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   After CRD is created:                                                 │
│                                                                         │
│   kubectl get databases          # Works!                               │
│   kubectl apply -f my-db.yaml    # Creates your custom resource        │
│   kubectl describe database mydb # Just like built-in resources        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. CRD Basics

### 1.1 What is a CRD?

| Term | Definition |
|------|------------|
| **CRD** | Custom Resource Definition - The schema/blueprint |
| **CR** | Custom Resource - An instance of a CRD |
| **Controller** | Code that watches CRs and takes action |
| **Operator** | CRD + Controller bundled together |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CRD vs CR vs Controller                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CRD (Definition/Schema)                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   "A Database resource has:                                     │   │
│   │    - name (string)                                              │   │
│   │    - engine (mysql/postgres)                                    │   │
│   │    - version (string)                                           │   │
│   │    - replicas (integer)"                                        │   │
│   │                                                                 │   │
│   │   apiVersion: apiextensions.k8s.io/v1                          │   │
│   │   kind: CustomResourceDefinition                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                        │                                                │
│                        │ Defines                                        │
│                        ▼                                                │
│   CR (Instance)                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   "Create a MySQL database called 'orders'                      │   │
│   │    with 3 replicas"                                             │   │
│   │                                                                 │   │
│   │   apiVersion: mycompany.io/v1                                  │   │
│   │   kind: Database                                                │   │
│   │   metadata:                                                     │   │
│   │     name: orders                                                │   │
│   │   spec:                                                         │   │
│   │     engine: mysql                                               │   │
│   │     replicas: 3                                                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                        │                                                │
│                        │ Watched by                                     │
│                        ▼                                                │
│   Controller (Automation)                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   "When a Database CR is created:                               │   │
│   │    1. Create a StatefulSet with replicas                        │   │
│   │    2. Create a Service for the database                         │   │
│   │    3. Create Secrets for credentials                            │   │
│   │    4. Monitor and update status"                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Why Use CRDs?

| Use Case | Example |
|----------|---------|
| **Simplify Complex Deployments** | `Database` CR instead of StatefulSet + PVC + Service + Secret |
| **Encode Domain Knowledge** | `Certificate` CR that auto-renews before expiry |
| **GitOps** | Declarative configuration for everything |
| **Platform Building** | Internal developer platforms |

---

## 2. Creating a CRD

### 2.1 Basic CRD Structure

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io     # plural.group
spec:
  group: mycompany.io              # API group
  names:
    kind: Database                  # Kind in YAML
    listKind: DatabaseList          # List resource type
    plural: databases               # kubectl get databases
    singular: database              # kubectl get database
    shortNames:                     # kubectl get db
    - db
  scope: Namespaced                 # or Cluster
  versions:
  - name: v1                        # API version
    served: true                    # Is this version served?
    storage: true                   # Is this the storage version?
    schema:
      openAPIV3Schema:              # Validation schema
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
              replicas:
                type: integer
```

### 2.2 Complete CRD Example

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    listKind: DatabaseList
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        required:
        - spec
        properties:
          spec:
            type: object
            required:
            - engine
            - version
            properties:
              engine:
                type: string
                enum:
                - mysql
                - postgres
                - mongodb
                description: Database engine type
              version:
                type: string
                description: Database version
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
                description: Number of replicas
              storage:
                type: string
                pattern: '^[0-9]+Gi$'
                default: "10Gi"
                description: Storage size (e.g., 10Gi)
          status:
            type: object
            properties:
              ready:
                type: boolean
              phase:
                type: string
              message:
                type: string
    # Additional columns in kubectl get
    additionalPrinterColumns:
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Ready
      type: boolean
      jsonPath: .status.ready
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
    # Enable status subresource
    subresources:
      status: {}
```

### 2.3 Apply the CRD

```bash
# Create the CRD
kubectl apply -f database-crd.yaml

# Verify CRD is created
kubectl get crd databases.example.com

# Now you can use the new resource type!
kubectl get databases
```

---

## 3. Creating Custom Resources

### 3.1 Creating a CR Instance

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: orders-db
  namespace: production
spec:
  engine: mysql
  version: "8.0"
  replicas: 3
  storage: "50Gi"
```

### 3.2 Apply and Manage CRs

```bash
# Create the custom resource
kubectl apply -f orders-db.yaml

# List all databases
kubectl get databases
kubectl get db                    # Using short name

# Describe the resource
kubectl describe database orders-db

# Get YAML
kubectl get database orders-db -o yaml

# Delete the resource
kubectl delete database orders-db
```

### 3.3 kubectl Output with Printer Columns

```bash
$ kubectl get databases

NAME        ENGINE   VERSION   REPLICAS   READY   AGE
orders-db   mysql    8.0       3          true    5m
users-db    postgres 15        2          false   2m
cache       mongodb  6.0       1          true    10m
```

---

## 4. Schema Validation

### 4.1 OpenAPI v3 Schema Types

| Type | Description | Example |
|------|-------------|---------|
| `string` | Text value | `"hello"` |
| `integer` | Whole number | `42` |
| `number` | Decimal number | `3.14` |
| `boolean` | true/false | `true` |
| `array` | List of items | `[1, 2, 3]` |
| `object` | Nested structure | `{key: value}` |

### 4.2 Validation Keywords

```yaml
properties:
  name:
    type: string
    minLength: 3
    maxLength: 63
    pattern: '^[a-z][a-z0-9-]*$'
  
  replicas:
    type: integer
    minimum: 1
    maximum: 100
    default: 3
  
  engine:
    type: string
    enum:
    - mysql
    - postgres
    - mongodb
  
  tags:
    type: array
    items:
      type: string
    minItems: 1
    maxItems: 10
  
  config:
    type: object
    additionalProperties:
      type: string
```

### 4.3 Required Fields

```yaml
spec:
  type: object
  required:
  - engine
  - version
  properties:
    engine:
      type: string
    version:
      type: string
    replicas:
      type: integer
      default: 1       # Optional with default
```

---

## 5. Subresources

### 5.1 Status Subresource

Enables `/status` endpoint for updating status separately:

```yaml
subresources:
  status: {}
```

Benefits:
- Controllers update `.status` without touching `.spec`
- Users can't accidentally modify `.status`
- Optimistic locking works correctly

```bash
# Update status (controller uses this)
kubectl patch database orders-db --subresource=status \
  --type=merge -p '{"status":{"ready":true}}'
```

### 5.2 Scale Subresource

Enables `kubectl scale` for custom resources:

```yaml
subresources:
  status: {}
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
    labelSelectorPath: .status.selector
```

```bash
# Now works!
kubectl scale database orders-db --replicas=5
```

---

## 6. Multiple Versions

### 6.1 API Versioning

```yaml
versions:
- name: v1
  served: true
  storage: true          # Storage version
  schema:
    openAPIV3Schema:
      # v1 schema
- name: v1beta1
  served: true
  storage: false         # Not storage version
  deprecated: true       # Mark as deprecated
  deprecationWarning: "v1beta1 is deprecated, use v1"
  schema:
    openAPIV3Schema:
      # v1beta1 schema
```

### 6.2 Version Priority

| Version | Priority | Example |
|---------|----------|---------|
| GA | Highest | v1, v2 |
| Beta | Medium | v1beta1, v2beta2 |
| Alpha | Lowest | v1alpha1 |

---

## 7. Controllers and Operators

### 7.1 What Controllers Do

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONTROLLER PATTERN                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   1. WATCH                                                      │   │
│   │      Controller watches for CR changes                          │   │
│   │      (create, update, delete)                                   │   │
│   │                                                                 │   │
│   │   2. RECONCILE                                                  │   │
│   │      For each CR, compare desired vs actual state              │   │
│   │                                                                 │   │
│   │   3. ACT                                                        │   │
│   │      Create/update/delete child resources                       │   │
│   │      (Pods, Services, Secrets, etc.)                           │   │
│   │                                                                 │   │
│   │   4. UPDATE STATUS                                              │   │
│   │      Report current state back to CR status                     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Example: Database Controller                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Database CR Created                                           │   │
│   │        │                                                        │   │
│   │        ▼                                                        │   │
│   │   Controller detects new Database                               │   │
│   │        │                                                        │   │
│   │        ├──▶ Create StatefulSet (for DB pods)                   │   │
│   │        ├──▶ Create Service (for connectivity)                  │   │
│   │        ├──▶ Create Secret (for credentials)                    │   │
│   │        ├──▶ Create PVC (for storage)                           │   │
│   │        │                                                        │   │
│   │        ▼                                                        │   │
│   │   Update Database status: ready=true                            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Popular Operator Frameworks

| Framework | Language | Description |
|-----------|----------|-------------|
| **Operator SDK** | Go, Ansible, Helm | Red Hat's framework |
| **Kubebuilder** | Go | Kubernetes SIG project |
| **Kopf** | Python | Python operators |
| **KUDO** | YAML | Declarative operators |
| **Metacontroller** | Any | Webhook-based |

### 7.3 CRD Without Controller

A CRD without a controller just stores data:

```yaml
# This CRD just stores configuration - no automation
apiVersion: config.mycompany.io/v1
kind: AppConfig
metadata:
  name: my-app-settings
spec:
  featureFlags:
    darkMode: true
    betaFeatures: false
  limits:
    maxUsers: 1000
```

Useful for:
- Storing configuration
- GitOps workflows
- Integration with external tools

---

## 8. Real-World CRD Examples

### 8.1 cert-manager Certificate

```yaml
# CRD provided by cert-manager
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-tls-cert
spec:
  secretName: my-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

### 8.2 Prometheus ServiceMonitor

```yaml
# CRD provided by prometheus-operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
```

### 8.3 Argo CD Application

```yaml
# CRD provided by Argo CD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

---

## 9. Command Reference

```bash
# CRD Commands
kubectl get crd                              # List all CRDs
kubectl get crd <name>                       # Get specific CRD
kubectl describe crd <name>                  # Describe CRD
kubectl delete crd <name>                    # Delete CRD (and all CRs!)

# Custom Resource Commands
kubectl get <plural>                         # List custom resources
kubectl get <plural> -A                      # All namespaces
kubectl describe <kind> <name>               # Describe CR
kubectl delete <kind> <name>                 # Delete CR
kubectl get <kind> <name> -o yaml            # Get YAML

# API Resources
kubectl api-resources                        # List all resource types
kubectl api-resources | grep mycompany       # Find your CRDs
kubectl api-versions                         # List API versions
```

---

## 10. Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CRD BEST PRACTICES                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Use a unique group (your domain, e.g., mycompany.io)        │   │
│   │  • Include validation schema                                   │   │
│   │  • Use status subresource                                      │   │
│   │  • Add printer columns for useful output                       │   │
│   │  • Version your CRDs (v1alpha1 → v1beta1 → v1)                │   │
│   │  • Document the CRD spec                                       │   │
│   │  • Handle deletion gracefully (finalizers)                     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Use generic group names (example.com)                       │   │
│   │  • Skip schema validation                                      │   │
│   │  • Delete CRD before cleaning up CRs                           │   │
│   │  • Store sensitive data in spec (use Secrets)                  │   │
│   │  • Create CRDs without controllers (usually)                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Understanding CRDs | 🟡 MEDIUM | Know what they are |
| kubectl with CRDs | 🟡 MEDIUM | get, describe custom resources |
| Creating CRDs | 🟢 LOW | Basic CRD YAML |

### Quick Reference

```bash
# List all CRDs
kubectl get crd

# Get custom resources
kubectl get <plural-name>

# Describe custom resource
kubectl describe <kind> <name>
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "List all custom resources of type Database"             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl get databases                                          │   │
│   │  kubectl get databases -A   # All namespaces                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "What CRDs are installed in the cluster?"                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl get crd                                                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Get details of a custom resource"                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe <kind> <name>                                 │   │
│   │  kubectl get <kind> <name> -o yaml                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CRD SUMMARY                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CRD = Custom Resource Definition (the schema/type)                    │
│   CR  = Custom Resource (instance of a CRD)                             │
│                                                                         │
│   CRD STRUCTURE:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  apiVersion: apiextensions.k8s.io/v1                           │   │
│   │  kind: CustomResourceDefinition                                │   │
│   │  metadata:                                                     │   │
│   │    name: databases.mycompany.io                                │   │
│   │  spec:                                                         │   │
│   │    group: mycompany.io                                         │   │
│   │    names:                                                      │   │
│   │      kind: Database                                            │   │
│   │      plural: databases                                         │   │
│   │    scope: Namespaced                                          │   │
│   │    versions:                                                   │   │
│   │    - name: v1                                                  │   │
│   │      schema: ...                                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   OPERATOR = CRD + Controller                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Logging & Monitoring** - Observability in Kubernetes
- Container logging
- kubectl logs
- Metrics and monitoring concepts

---

**Chapter 25 Complete! 🎉**

You now understand:
- What CRDs are and why they're used
- Creating Custom Resource Definitions
- Schema validation
- Subresources (status, scale)
- Controllers and Operators
- Real-world CRD examples
- CKA exam preparation

