# Chapter 18: RBAC - Role-Based Access Control

## Introduction

In any production system, you need to control **who can do what**. Kubernetes uses Role-Based Access Control (RBAC) to manage permissions. RBAC lets you define fine-grained access policies - for example, allowing developers to view pods but not delete them, or letting a CI/CD service account deploy to specific namespaces only.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE ACCESS CONTROL CHALLENGE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without RBAC (Dangerous):                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Everyone has full cluster access                              │   │
│   │                                                                 │   │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │   │
│   │   │  Dev    │  │   QA    │  │ Jenkins │  │  Intern │          │   │
│   │   │         │  │         │  │         │  │         │          │   │
│   │   │ ADMIN   │  │ ADMIN   │  │ ADMIN   │  │ ADMIN   │  ← YIKES!│   │
│   │   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘          │   │
│   │        │            │            │            │                │   │
│   │        └────────────┴────────────┴────────────┘                │   │
│   │                         │                                       │   │
│   │                         ▼                                       │   │
│   │              ┌───────────────────┐                              │   │
│   │              │  FULL CLUSTER     │  Can delete anything!       │   │
│   │              │  ACCESS           │  Can read all secrets!       │   │
│   │              └───────────────────┘  Can modify RBAC!            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With RBAC (Secure):                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Fine-grained permissions per user/service                     │   │
│   │                                                                 │   │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │   │
│   │   │  Dev    │  │   QA    │  │ Jenkins │  │  Intern │          │   │
│   │   │         │  │         │  │         │  │         │          │   │
│   │   │view pods│  │view all │  │deploy   │  │view only│          │   │
│   │   │edit deps│  │exec pods│  │ns: prod │  │dev ns   │          │   │
│   │   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘          │   │
│   │        │            │            │            │                │   │
│   │        ▼            ▼            ▼            ▼                │   │
│   │   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐           │   │
│   │   │dev-role│   │qa-role │   │ci-role │   │readonly│           │   │
│   │   └────────┘   └────────┘   └────────┘   └────────┘           │   │
│   │                                                                 │   │
│   │   Principle of Least Privilege ✓                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Authentication vs Authorization

### 1.1 The Two Steps of Access Control

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 AUTHENTICATION vs AUTHORIZATION                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   User/Pod makes API request                                    │   │
│   │         │                                                       │   │
│   │         ▼                                                       │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  STEP 1: AUTHENTICATION (AuthN)                         │   │   │
│   │   │                                                         │   │   │
│   │   │  "WHO ARE YOU?"                                         │   │   │
│   │   │                                                         │   │   │
│   │   │  Methods:                                               │   │   │
│   │   │  • X.509 Client Certificates                            │   │   │
│   │   │  • Bearer Tokens (Service Accounts)                     │   │   │
│   │   │  • OpenID Connect (OIDC)                                │   │   │
│   │   │  • Webhook Token Authentication                         │   │   │
│   │   │                                                         │   │   │
│   │   │  Result: Identity established (user, group, SA)         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │         │                                                       │   │
│   │         ▼  Identity verified                                    │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  STEP 2: AUTHORIZATION (AuthZ)                          │   │   │
│   │   │                                                         │   │   │
│   │   │  "WHAT CAN YOU DO?"                                     │   │   │
│   │   │                                                         │   │   │
│   │   │  Methods:                                               │   │   │
│   │   │  • RBAC (Role-Based Access Control) ← Most common       │   │   │
│   │   │  • ABAC (Attribute-Based Access Control)                │   │   │
│   │   │  • Webhook                                              │   │   │
│   │   │  • Node Authorization                                   │   │   │
│   │   │                                                         │   │   │
│   │   │  Result: Allow or Deny the request                      │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │         │                                                       │   │
│   │         ▼  Authorized                                           │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  STEP 3: ADMISSION CONTROL                              │   │   │
│   │   │                                                         │   │   │
│   │   │  Additional validations and mutations                   │   │   │
│   │   │  (Pod Security, Resource Quotas, etc.)                  │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │         │                                                       │   │
│   │         ▼                                                       │   │
│   │   Request processed by API server                               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Identity Types in Kubernetes

| Identity Type | Description | How Created | Use Case |
|---------------|-------------|-------------|----------|
| **User** | Human operator | External (certs, OIDC) | kubectl access |
| **Group** | Collection of users | External | Team permissions |
| **ServiceAccount** | Pod identity | Kubernetes object | Pod-to-API access |

### 1.3 Important Built-in Groups

| Group | Description |
|-------|-------------|
| `system:unauthenticated` | Requests without valid credentials |
| `system:authenticated` | All authenticated users |
| `system:masters` | Full cluster admin (superuser) |
| `system:serviceaccounts` | All service accounts |
| `system:serviceaccounts:<ns>` | Service accounts in namespace |

---

## 2. RBAC API Objects

### 2.1 The Four RBAC Objects

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      RBAC OBJECT MODEL                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    WHAT PERMISSIONS?                            │   │
│   │                                                                 │   │
│   │   ┌─────────────────────┐    ┌─────────────────────┐           │   │
│   │   │       ROLE          │    │    CLUSTERROLE      │           │   │
│   │   │                     │    │                     │           │   │
│   │   │  Namespaced         │    │  Cluster-wide       │           │   │
│   │   │  permissions        │    │  permissions        │           │   │
│   │   │                     │    │                     │           │   │
│   │   │  e.g., pods in      │    │  e.g., nodes,       │           │   │
│   │   │  "dev" namespace    │    │  PVs, namespaces    │           │   │
│   │   └─────────────────────┘    └─────────────────────┘           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    │  Bound via                         │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    WHO GETS PERMISSIONS?                        │   │
│   │                                                                 │   │
│   │   ┌─────────────────────┐    ┌─────────────────────┐           │   │
│   │   │    ROLEBINDING      │    │ CLUSTERROLEBINDING  │           │   │
│   │   │                     │    │                     │           │   │
│   │   │  Grants Role or     │    │  Grants ClusterRole │           │   │
│   │   │  ClusterRole in     │    │  cluster-wide       │           │   │
│   │   │  ONE namespace      │    │                     │           │   │
│   │   │                     │    │                     │           │   │
│   │   │  to Users, Groups,  │    │  to Users, Groups,  │           │   │
│   │   │  or ServiceAccounts │    │  or ServiceAccounts │           │   │
│   │   └─────────────────────┘    └─────────────────────┘           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Summary:                                                              │
│   • Role + RoleBinding = Namespace-scoped permissions                   │
│   • ClusterRole + ClusterRoleBinding = Cluster-wide permissions         │
│   • ClusterRole + RoleBinding = Reusable role in specific namespace    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 RBAC Combinations

| Role Type | Binding Type | Scope | Use Case |
|-----------|--------------|-------|----------|
| Role | RoleBinding | Single namespace | Dev access to dev namespace |
| ClusterRole | RoleBinding | Single namespace | Reuse ClusterRole in namespace |
| ClusterRole | ClusterRoleBinding | Entire cluster | Cluster admin, node access |

---

## 3. Roles and ClusterRoles

### 3.1 Role (Namespace-Scoped)

A **Role** defines permissions within a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev                      # Scoped to this namespace
rules:
- apiGroups: [""]                     # "" = core API group
  resources: ["pods"]                 # Resource types
  verbs: ["get", "list", "watch"]     # Allowed actions
```

### 3.2 ClusterRole (Cluster-Scoped)

A **ClusterRole** defines permissions across the entire cluster or for cluster-scoped resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader                   # No namespace (cluster-wide)
rules:
- apiGroups: [""]
  resources: ["nodes"]                # Cluster-scoped resource
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]                 # Can also include namespaced resources
  verbs: ["get", "list"]
```

### 3.3 Understanding Rules

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RULE ANATOMY                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   rules:                                                                │
│   - apiGroups: [""]              ◀── API Group                         │
│     resources: ["pods"]          ◀── Resource Type                     │
│     verbs: ["get", "list"]       ◀── Actions                           │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  API GROUPS                                                     │   │
│   ├─────────────────────────────────────────────────────────────────┤   │
│   │  ""              │ Core API (pods, services, secrets, etc.)     │   │
│   │  "apps"          │ Deployments, DaemonSets, StatefulSets        │   │
│   │  "batch"         │ Jobs, CronJobs                               │   │
│   │  "networking.k8s.io" │ NetworkPolicies, Ingresses              │   │
│   │  "rbac.authorization.k8s.io" │ Roles, RoleBindings             │   │
│   │  "storage.k8s.io" │ StorageClasses, CSI                        │   │
│   │  "*"             │ All API groups                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  VERBS (Actions)                                                │   │
│   ├─────────────────────────────────────────────────────────────────┤   │
│   │  get      │ Read a single resource                              │   │
│   │  list     │ List resources                                      │   │
│   │  watch    │ Watch for changes (streaming)                       │   │
│   │  create   │ Create new resources                                │   │
│   │  update   │ Update existing resources                           │   │
│   │  patch    │ Partially update resources                          │   │
│   │  delete   │ Delete a single resource                            │   │
│   │  deletecollection │ Delete multiple resources                   │   │
│   │  *        │ All verbs                                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  COMMON RESOURCES                                               │   │
│   ├─────────────────────────────────────────────────────────────────┤   │
│   │  Core (""): pods, services, secrets, configmaps, endpoints,    │   │
│   │            persistentvolumeclaims, namespaces, nodes            │   │
│   │                                                                 │   │
│   │  apps: deployments, daemonsets, statefulsets, replicasets      │   │
│   │                                                                 │   │
│   │  batch: jobs, cronjobs                                         │   │
│   │                                                                 │   │
│   │  *: All resources in the API group                              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Role Examples

#### Read-Only Access to Pods

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]     # Include pod logs
  verbs: ["get", "list", "watch"]
```

#### Developer Role (Common Pattern)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
# View most resources
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "configmaps", "jobs"]
  verbs: ["get", "list", "watch"]
# Manage deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "update", "patch", "delete"]
# Execute into pods for debugging
- apiGroups: [""]
  resources: ["pods/exec", "pods/log"]
  verbs: ["create", "get"]
# Port-forward for debugging
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
```

#### Full Admin Within Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: dev
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### 3.5 ClusterRole Examples

#### Node Reader (Cluster-Scoped Resource)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

#### Cluster-Wide Pod Viewer

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]              # Need to list namespaces too
```

#### Secret Reader (Dangerous - Use Carefully)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
# ⚠️ This grants access to ALL secrets cluster-wide!
```

### 3.6 Built-in ClusterRoles

Kubernetes comes with several built-in ClusterRoles:

| ClusterRole | Description | Use Case |
|-------------|-------------|----------|
| `cluster-admin` | Full access to everything | Super admin |
| `admin` | Full access within namespace | Namespace admin |
| `edit` | Read/write most resources | Developers |
| `view` | Read-only access | Viewers, auditors |

```bash
# List all built-in ClusterRoles
kubectl get clusterroles | grep -E "^(cluster-admin|admin|edit|view)$"

# View details of built-in role
kubectl describe clusterrole view
```

---

## 4. RoleBindings and ClusterRoleBindings

### 4.1 RoleBinding

A **RoleBinding** grants permissions defined in a Role to subjects within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-readers
  namespace: dev                      # Binding is namespaced
subjects:
- kind: User
  name: alice                         # User name
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers                    # Group name
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-app-sa                     # ServiceAccount name
  namespace: dev                      # SA namespace
roleRef:
  kind: Role                          # or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4.2 ClusterRoleBinding

A **ClusterRoleBinding** grants cluster-wide permissions.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding         # No namespace (cluster-wide)
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: cluster-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 4.3 Subject Types

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SUBJECT TYPES                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  USER                                                           │   │
│   │  ────────────────────────────────────────────────────────────   │   │
│   │  subjects:                                                      │   │
│   │  - kind: User                                                   │   │
│   │    name: alice@example.com    # From certificate CN or OIDC     │   │
│   │    apiGroup: rbac.authorization.k8s.io                          │   │
│   │                                                                 │   │
│   │  Notes:                                                         │   │
│   │  • Not a Kubernetes object (external identity)                  │   │
│   │  • Comes from certificate CN, OIDC, etc.                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  GROUP                                                          │   │
│   │  ────────────────────────────────────────────────────────────   │   │
│   │  subjects:                                                      │   │
│   │  - kind: Group                                                  │   │
│   │    name: developers           # From certificate O or OIDC     │   │
│   │    apiGroup: rbac.authorization.k8s.io                          │   │
│   │                                                                 │   │
│   │  Notes:                                                         │   │
│   │  • Not a Kubernetes object (external identity)                  │   │
│   │  • Prefix "system:" for built-in groups                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  SERVICEACCOUNT                                                 │   │
│   │  ────────────────────────────────────────────────────────────   │   │
│   │  subjects:                                                      │   │
│   │  - kind: ServiceAccount                                         │   │
│   │    name: my-app-sa            # Kubernetes object               │   │
│   │    namespace: dev             # Required for SA                 │   │
│   │                                                                 │   │
│   │  Notes:                                                         │   │
│   │  • IS a Kubernetes object                                       │   │
│   │  • Needs namespace specified                                    │   │
│   │  • Used by pods for API access                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Binding Patterns

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     COMMON BINDING PATTERNS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Pattern 1: Role + RoleBinding (Single Namespace)                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌────────────┐      ┌─────────────┐      ┌────────────┐      │   │
│   │   │    Role    │◀─────│ RoleBinding │─────▶│   User     │      │   │
│   │   │  (in dev)  │      │  (in dev)   │      │  (alice)   │      │   │
│   │   └────────────┘      └─────────────┘      └────────────┘      │   │
│   │                                                                 │   │
│   │   Result: alice can do X in dev namespace only                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Pattern 2: ClusterRole + RoleBinding (Reusable Role)                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌────────────┐      ┌─────────────┐      ┌────────────┐      │   │
│   │   │ClusterRole │◀─────│ RoleBinding │─────▶│   User     │      │   │
│   │   │ (pod-view) │      │  (in dev)   │      │  (alice)   │      │   │
│   │   └────────────┘      └─────────────┘      └────────────┘      │   │
│   │        ▲                                                        │   │
│   │        │              ┌─────────────┐      ┌────────────┐      │   │
│   │        └──────────────│ RoleBinding │─────▶│   User     │      │   │
│   │                       │ (in prod)   │      │   (bob)    │      │   │
│   │                       └─────────────┘      └────────────┘      │   │
│   │                                                                 │   │
│   │   Result: Same ClusterRole used in multiple namespaces          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Pattern 3: ClusterRole + ClusterRoleBinding (Cluster-Wide)            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌────────────┐      ┌─────────────────┐    ┌────────────┐    │   │
│   │   │ClusterRole │◀─────│ClusterRoleBinding│───▶│   User     │    │   │
│   │   │(cluster-   │      │                  │    │  (admin)   │    │   │
│   │   │   admin)   │      └─────────────────┘    └────────────┘    │   │
│   │   └────────────┘                                                │   │
│   │                                                                 │   │
│   │   Result: admin can do X across ALL namespaces                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Service Accounts

### 5.1 What is a Service Account?

A **ServiceAccount** provides an identity for pods. It's how pods authenticate to the Kubernetes API.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SERVICE ACCOUNT OVERVIEW                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                          POD                                    │   │
│   │                                                                 │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                     Container                           │   │   │
│   │   │                                                         │   │   │
│   │   │   /var/run/secrets/kubernetes.io/serviceaccount/        │   │   │
│   │   │   ├── token         ◀── JWT token for API auth          │   │   │
│   │   │   ├── ca.crt        ◀── Cluster CA certificate          │   │   │
│   │   │   └── namespace     ◀── Pod's namespace                 │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   spec:                                                         │   │
│   │     serviceAccountName: my-app-sa                               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              │  Uses token to authenticate              │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     kube-apiserver                              │   │
│   │                                                                 │   │
│   │   1. Validates token                                            │   │
│   │   2. Identifies ServiceAccount                                  │   │
│   │   3. Checks RBAC permissions                                    │   │
│   │   4. Allows/Denies request                                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Default Service Account

Every namespace has a `default` ServiceAccount. Pods use it unless specified otherwise.

```bash
# List service accounts
kubectl get serviceaccounts
kubectl get sa                        # Short form

# View default SA
kubectl get sa default -o yaml
```

### 5.3 Creating Service Accounts

#### Imperative

```bash
kubectl create serviceaccount my-app-sa
kubectl create sa my-app-sa -n dev
```

#### Declarative

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: dev
  labels:
    app: my-app
automountServiceAccountToken: true    # Default is true
```

### 5.4 Using Service Account in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa       # Use this SA
  containers:
  - name: app
    image: myapp
```

### 5.5 Disabling Token Mounting

For pods that don't need API access:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-api-access
spec:
  automountServiceAccountToken: false   # Don't mount token
  containers:
  - name: app
    image: myapp
```

### 5.6 Complete ServiceAccount + RBAC Example

```yaml
# 1. Create ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-manager
  namespace: dev
---
# 2. Create Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
---
# 3. Create RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: pod-manager
  namespace: dev
roleRef:
  kind: Role
  name: pod-manager-role
  apiGroup: rbac.authorization.k8s.io
---
# 4. Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: pod-manager-app
  namespace: dev
spec:
  serviceAccountName: pod-manager
  containers:
  - name: app
    image: bitnami/kubectl
    command: ['sleep', '3600']
```

---

## 6. Advanced RBAC Features

### 6.1 Resource Names (Specific Resources)

Limit access to specific named resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: specific-configmap-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config", "db-config"]   # Only these CMs
  verbs: ["get"]
```

### 6.2 Subresources

Some resources have subresources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-debug
  namespace: dev
rules:
# Exec into pods
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
# View pod logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
# Port-forward
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
# Deployment scale
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get", "update", "patch"]
```

### 6.3 Common Subresources

| Resource | Subresource | Description |
|----------|-------------|-------------|
| pods | exec | Execute commands in container |
| pods | log | View pod logs |
| pods | portforward | Port forwarding |
| pods | attach | Attach to container |
| deployments | scale | Scale deployment |
| deployments | status | Deployment status |
| nodes | proxy | Node proxy |

### 6.4 Non-Resource URLs

For health checks and metrics:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: health-checker
rules:
- nonResourceURLs: ["/healthz", "/readyz", "/livez"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

### 6.5 Aggregated ClusterRoles

Combine multiple roles using labels:

```yaml
# Aggregate all matching roles
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # Rules are automatically filled in
---
# This gets aggregated into "monitoring"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

---

## 7. Testing and Debugging RBAC

### 7.1 Can I? (kubectl auth can-i)

```bash
# Check if YOU can do something
kubectl auth can-i create pods
kubectl auth can-i delete deployments -n prod
kubectl auth can-i '*' '*'                    # Am I cluster-admin?

# Check if a USER can do something
kubectl auth can-i get pods --as alice
kubectl auth can-i delete deployments --as bob -n dev

# Check if a SERVICE ACCOUNT can do something
kubectl auth can-i list secrets \
  --as system:serviceaccount:dev:my-app-sa

# List all permissions for current user
kubectl auth can-i --list
kubectl auth can-i --list -n dev
```

### 7.2 Who Can? (kubectl auth who-can)

```bash
# Who can delete pods in current namespace?
kubectl auth who-can delete pods

# Who can create secrets cluster-wide?
kubectl auth who-can create secrets --all-namespaces
```

### 7.3 Debugging Access Denied

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEBUGGING ACCESS DENIED                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Error: "pods is forbidden: User 'alice' cannot get pods"              │
│                                                                         │
│   Debugging Steps:                                                      │
│                                                                         │
│   1. CHECK IDENTITY                                                     │
│      ┌─────────────────────────────────────────────────────────────┐   │
│      │  kubectl auth whoami                                         │   │
│      │  # Shows your current identity                               │   │
│      └─────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   2. CHECK WHAT YOU CAN DO                                              │
│      ┌─────────────────────────────────────────────────────────────┐   │
│      │  kubectl auth can-i get pods                                 │   │
│      │  kubectl auth can-i --list                                   │   │
│      └─────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   3. FIND ROLEBINDINGS                                                  │
│      ┌─────────────────────────────────────────────────────────────┐   │
│      │  kubectl get rolebindings -A | grep alice                    │   │
│      │  kubectl get clusterrolebindings | grep alice                │   │
│      └─────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   4. CHECK ROLE DETAILS                                                 │
│      ┌─────────────────────────────────────────────────────────────┐   │
│      │  kubectl describe role <role-name>                           │   │
│      │  kubectl describe clusterrole <role-name>                    │   │
│      └─────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   5. VERIFY API GROUP & RESOURCE                                        │
│      ┌─────────────────────────────────────────────────────────────┐   │
│      │  kubectl api-resources | grep pods                           │   │
│      │  # Check correct apiGroup and resource name                  │   │
│      └─────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. RBAC Best Practices

### 8.1 Principle of Least Privilege

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   LEAST PRIVILEGE PRINCIPLE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Start with NO permissions, add as needed                     │   │
│   │  • Use namespace-scoped Roles when possible                     │   │
│   │  • Create separate ServiceAccounts per app                      │   │
│   │  • Avoid wildcards (*) in production                            │   │
│   │  • Review and audit permissions regularly                       │   │
│   │  • Use resourceNames to limit to specific resources             │   │
│   │  • Disable automountServiceAccountToken when not needed         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Give cluster-admin to everyone                               │   │
│   │  • Use default service account for all apps                     │   │
│   │  • Grant secrets access without need                            │   │
│   │  • Use ClusterRoleBinding when RoleBinding suffices             │   │
│   │  • Leave unused bindings                                        │   │
│   │  • Grant escalate, bind, or impersonate without thought         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Role Hierarchy Recommendations

| Role Level | Scope | Example Users |
|------------|-------|---------------|
| `cluster-admin` | Full cluster | Platform team (limited) |
| `admin` (per ns) | Full namespace | Team leads |
| `edit` (per ns) | Create/update | Developers |
| `view` (per ns) | Read-only | Auditors, support |
| Custom roles | Specific needs | CI/CD, operators |

### 8.3 ServiceAccount Best Practices

```yaml
# Good: Dedicated SA with minimal permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-sa   # Not default!
      automountServiceAccountToken: false  # If no API access needed
      containers:
      - name: app
        image: myapp
```

---

## 9. Command Reference

### Role Commands

```bash
# Create role imperatively
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

# List roles
kubectl get roles -n dev

# Describe role
kubectl describe role pod-reader -n dev

# Delete role
kubectl delete role pod-reader -n dev
```

### ClusterRole Commands

```bash
# Create clusterrole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# List clusterroles
kubectl get clusterroles

# Describe clusterrole
kubectl describe clusterrole view
```

### RoleBinding Commands

```bash
# Create rolebinding
kubectl create rolebinding dev-pod-readers \
  --role=pod-reader \
  --user=alice \
  --group=developers \
  --serviceaccount=dev:my-sa \
  -n dev

# List rolebindings
kubectl get rolebindings -n dev

# Describe rolebinding
kubectl describe rolebinding dev-pod-readers -n dev
```

### ClusterRoleBinding Commands

```bash
# Create clusterrolebinding
kubectl create clusterrolebinding admin-binding \
  --clusterrole=cluster-admin \
  --user=admin@example.com

# List clusterrolebindings
kubectl get clusterrolebindings

# Delete clusterrolebinding
kubectl delete clusterrolebinding admin-binding
```

### ServiceAccount Commands

```bash
# Create service account
kubectl create sa my-app-sa -n dev

# List service accounts
kubectl get sa -n dev

# Get SA token (K8s 1.24+)
kubectl create token my-app-sa -n dev

# View SA details
kubectl describe sa my-app-sa -n dev
```

---

## 10. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Create Role/ClusterRole | 🔴 HIGH | YAML from scratch |
| Create RoleBinding | 🔴 HIGH | Link role to subject |
| ServiceAccount | 🔴 HIGH | Create and use in pod |
| kubectl auth can-i | 🔴 HIGH | Debug permissions |
| Built-in roles | 🟡 MEDIUM | Know view, edit, admin |

### Quick Reference for Exam

```yaml
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: dev
roleRef:
  kind: Role
  name: my-role
  apiGroup: rbac.authorization.k8s.io
```

### Imperative Commands (Fast for Exam)

```bash
# Create role
kubectl create role pod-reader \
  --verb=get,list \
  --resource=pods \
  -n dev

# Create rolebinding
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=dev:my-sa \
  -n dev

# Create clusterrole
kubectl create clusterrole node-reader \
  --verb=get,list \
  --resource=nodes

# Create clusterrolebinding
kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=alice

# Create serviceaccount
kubectl create sa my-sa -n dev
```

### Common Exam Mistakes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA MISTAKES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ❌ Mistake 1: Wrong API group                                         │
│      Core resources ("") vs apps, batch, networking.k8s.io             │
│      Use: kubectl api-resources | grep <resource>                      │
│                                                                         │
│   ❌ Mistake 2: Forgetting namespace for ServiceAccount subject         │
│      subjects:                                                          │
│      - kind: ServiceAccount                                             │
│        name: my-sa                                                      │
│        namespace: dev    ◀── Required!                                 │
│                                                                         │
│   ❌ Mistake 3: Using Role when ClusterRole needed                      │
│      Cluster-scoped resources (nodes, PVs) need ClusterRole            │
│                                                                         │
│   ❌ Mistake 4: RoleBinding can reference ClusterRole                   │
│      But it only grants permissions in that namespace                   │
│                                                                         │
│   ❌ Mistake 5: Forgetting verbs                                        │
│      "list" ≠ "get", you might need both                               │
│                                                                         │
│   ❌ Mistake 6: Wrong subresource syntax                                │
│      resources: ["pods/log"] not resources: ["pods", "log"]            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Docker to Kubernetes Mapping

| Docker Concept | Kubernetes RBAC Equivalent |
|---------------|---------------------------|
| Docker group membership | Service Account + RoleBinding |
| Docker socket access | ServiceAccount with pod permissions |
| Docker registry auth | imagePullSecrets |
| Docker compose services | ServiceAccount per deployment |
| No Docker RBAC (root or not) | Fine-grained RBAC |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RBAC DECISION TREE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   What permissions do you need?                                         │
│       │                                                                 │
│       ├──▶ Cluster-scoped resources (nodes, PVs)?                      │
│       │       │                                                         │
│       │       └──▶ Use ClusterRole                                     │
│       │                                                                 │
│       └──▶ Namespace-scoped resources?                                 │
│               │                                                         │
│               ├──▶ Same permissions in multiple namespaces?            │
│               │       │                                                 │
│               │       └──▶ Use ClusterRole + RoleBinding (each ns)     │
│               │                                                         │
│               └──▶ Single namespace only?                              │
│                       │                                                 │
│                       └──▶ Use Role + RoleBinding                      │
│                                                                         │
│   Who needs the permissions?                                            │
│       │                                                                 │
│       ├──▶ Human user → subjects.kind: User                           │
│       │                                                                 │
│       ├──▶ Team/group → subjects.kind: Group                          │
│       │                                                                 │
│       └──▶ Pod/application → subjects.kind: ServiceAccount            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Security Contexts** - Pod and container security settings
- Running as non-root
- Capabilities
- SELinux/AppArmor
- Seccomp profiles

---

**Chapter 18 Complete! 🎉**

You now understand:
- Authentication vs Authorization
- RBAC API objects (Role, ClusterRole, RoleBinding, ClusterRoleBinding)
- Service Accounts for pod identity
- Verbs, resources, and API groups
- Testing with kubectl auth can-i
- Best practices for least privilege
- CKA exam preparation

