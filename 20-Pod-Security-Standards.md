# Chapter 20: Pod Security Standards - Cluster-Wide Security Enforcement

## Introduction

In the previous chapter, we learned how to configure security contexts for individual pods. But how do you **enforce** that all pods in your cluster follow security best practices? That's where **Pod Security Standards** and **Pod Security Admission** come in.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE ENFORCEMENT PROBLEM                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Pod Security Enforcement:                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Developer A:                Developer B:                      │   │
│   │   ┌─────────────────┐        ┌─────────────────┐               │   │
│   │   │ Secure Pod      │        │ Insecure Pod    │               │   │
│   │   │                 │        │                 │               │   │
│   │   │ runAsNonRoot:   │        │ privileged: true│               │   │
│   │   │   true          │        │ hostNetwork:true│               │   │
│   │   │ readOnlyRoot:   │        │ capabilities:   │               │   │
│   │   │   true          │        │   - SYS_ADMIN   │               │   │
│   │   └─────────────────┘        └─────────────────┘               │   │
│   │         ✅                          ✅                          │   │
│   │   Both accepted by Kubernetes!                                  │   │
│   │                                                                 │   │
│   │   ❌ No enforcement = security is optional                     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Pod Security Admission:                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Developer A:                Developer B:                      │   │
│   │   ┌─────────────────┐        ┌─────────────────┐               │   │
│   │   │ Secure Pod      │        │ Insecure Pod    │               │   │
│   │   │                 │        │                 │               │   │
│   │   │ runAsNonRoot:   │        │ privileged: true│               │   │
│   │   │   true          │        │ (BLOCKED!)      │               │   │
│   │   └─────────────────┘        └─────────────────┘               │   │
│   │         ✅                          ❌                          │   │
│   │   Accepted!                  REJECTED by PSA!                   │   │
│   │                                                                 │   │
│   │   ✅ Enforcement = security is mandatory                       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Pod Security Standards (PSS)

### 1.1 What are Pod Security Standards?

**Pod Security Standards** define three security profiles that cover a broad spectrum of security needs:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THREE SECURITY PROFILES                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      PRIVILEGED                                 │   │
│   │                                                                 │   │
│   │   No restrictions at all                                        │   │
│   │                                                                 │   │
│   │   • Allows everything                                           │   │
│   │   • For system-level workloads                                  │   │
│   │   • CNI plugins, storage drivers                                │   │
│   │   • Trust level: Full trust                                     │   │
│   │                                                                 │   │
│   │   Use for: kube-system, monitoring agents                       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                       BASELINE                                  │   │
│   │                                                                 │   │
│   │   Prevents known privilege escalations                          │   │
│   │                                                                 │   │
│   │   Blocks:                                                       │   │
│   │   • privileged: true                                            │   │
│   │   • hostNetwork/hostPID/hostIPC                                │   │
│   │   • Dangerous capabilities                                      │   │
│   │   • Host path volumes                                           │   │
│   │                                                                 │   │
│   │   Allows:                                                       │   │
│   │   • Running as root                                             │   │
│   │   • Most capabilities                                           │   │
│   │   • Writable root filesystem                                    │   │
│   │                                                                 │   │
│   │   Use for: Most applications (easy adoption)                    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      RESTRICTED                                 │   │
│   │                                                                 │   │
│   │   Maximum security, current hardening best practices            │   │
│   │                                                                 │   │
│   │   Requires:                                                     │   │
│   │   • runAsNonRoot: true                                          │   │
│   │   • allowPrivilegeEscalation: false                             │   │
│   │   • Drop ALL capabilities                                       │   │
│   │   • Seccomp profile                                             │   │
│   │                                                                 │   │
│   │   Blocks:                                                       │   │
│   │   • Everything Baseline blocks PLUS                             │   │
│   │   • Running as root                                             │   │
│   │   • Adding any capabilities                                     │   │
│   │   • No seccomp profile                                          │   │
│   │                                                                 │   │
│   │   Use for: Highly sensitive workloads                           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Profile Comparison

| Control | Privileged | Baseline | Restricted |
|---------|------------|----------|------------|
| `privileged` | ✅ Allowed | ❌ Must be false | ❌ Must be false |
| `hostNetwork` | ✅ Allowed | ❌ Must be false | ❌ Must be false |
| `hostPID` | ✅ Allowed | ❌ Must be false | ❌ Must be false |
| `hostIPC` | ✅ Allowed | ❌ Must be false | ❌ Must be false |
| `hostPath` volumes | ✅ Allowed | ❌ Not allowed | ❌ Not allowed |
| `hostPorts` | ✅ Allowed | ⚠️ Restricted | ❌ Not allowed |
| Dangerous capabilities | ✅ Allowed | ❌ Not allowed | ❌ Not allowed |
| `runAsNonRoot` | ✅ Allowed | ✅ Not required | ✅ **Required** |
| `allowPrivilegeEscalation` | ✅ Allowed | ✅ Not required | ❌ Must be false |
| `capabilities.drop` | ✅ Not required | ✅ Not required | ✅ Must drop ALL |
| Seccomp profile | ✅ Not required | ✅ Not required | ✅ **Required** |
| Running as root | ✅ Allowed | ✅ Allowed | ❌ Not allowed |

---

## 2. Pod Security Admission (PSA)

### 2.1 What is Pod Security Admission?

**Pod Security Admission** (PSA) is a built-in Kubernetes admission controller that enforces Pod Security Standards at the namespace level using labels.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   POD SECURITY ADMISSION FLOW                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   kubectl apply -f pod.yaml                                     │   │
│   │         │                                                       │   │
│   │         ▼                                                       │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │              kube-apiserver                             │   │   │
│   │   │                                                         │   │   │
│   │   │   1. Authentication ✓                                   │   │   │
│   │   │   2. Authorization (RBAC) ✓                             │   │   │
│   │   │   3. Admission Controllers:                             │   │   │
│   │   │      │                                                  │   │   │
│   │   │      ▼                                                  │   │   │
│   │   │   ┌─────────────────────────────────────────────────┐   │   │   │
│   │   │   │        Pod Security Admission                   │   │   │   │
│   │   │   │                                                 │   │   │   │
│   │   │   │  Check namespace labels:                        │   │   │   │
│   │   │   │  pod-security.kubernetes.io/enforce: baseline   │   │   │   │
│   │   │   │                                                 │   │   │   │
│   │   │   │  Does pod comply with BASELINE?                 │   │   │   │
│   │   │   │                                                 │   │   │   │
│   │   │   │  ├── YES → Allow pod                           │   │   │   │
│   │   │   │  └── NO  → Reject pod (if enforce mode)        │   │   │   │
│   │   │   │                                                 │   │   │   │
│   │   │   └─────────────────────────────────────────────────┘   │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Enforcement Modes

PSA supports three modes for each profile:

| Mode | Behavior | Impact |
|------|----------|--------|
| **enforce** | Violations **block** pod creation | Pod rejected |
| **audit** | Violations **logged** to audit log | Pod allowed, logged |
| **warn** | Violations return **warning** to user | Pod allowed, warning shown |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ENFORCEMENT MODES                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ENFORCE MODE                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   $ kubectl apply -f insecure-pod.yaml                         │   │
│   │                                                                 │   │
│   │   Error: pods "insecure-pod" is forbidden:                     │   │
│   │   violates PodSecurity "baseline:latest":                      │   │
│   │   privileged (container "app" must not set                     │   │
│   │   securityContext.privileged=true)                             │   │
│   │                                                                 │   │
│   │   Result: Pod NOT created ❌                                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   WARN MODE                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   $ kubectl apply -f insecure-pod.yaml                         │   │
│   │                                                                 │   │
│   │   Warning: would violate PodSecurity "baseline:latest":        │   │
│   │   privileged (container "app" sets                             │   │
│   │   securityContext.privileged=true)                             │   │
│   │                                                                 │   │
│   │   pod/insecure-pod created                                     │   │
│   │                                                                 │   │
│   │   Result: Pod created ✅ but warning shown ⚠️                  │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   AUDIT MODE                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   $ kubectl apply -f insecure-pod.yaml                         │   │
│   │                                                                 │   │
│   │   pod/insecure-pod created                                     │   │
│   │                                                                 │   │
│   │   Result: Pod created ✅ silently                              │   │
│   │   (Violation logged to audit log for review)                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Gradual Rollout Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   RECOMMENDED ROLLOUT STRATEGY                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Phase 1: Audit                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   pod-security.kubernetes.io/audit: restricted                  │   │
│   │                                                                 │   │
│   │   • Log violations without blocking                             │   │
│   │   • Review audit logs to find non-compliant pods               │   │
│   │   • No impact on running workloads                              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│   Phase 2: Warn                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   pod-security.kubernetes.io/audit: restricted                  │   │
│   │   pod-security.kubernetes.io/warn: restricted                   │   │
│   │                                                                 │   │
│   │   • Continue auditing                                           │   │
│   │   • Show warnings to developers                                 │   │
│   │   • Teams fix their deployments                                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│   Phase 3: Enforce                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   pod-security.kubernetes.io/enforce: restricted                │   │
│   │   pod-security.kubernetes.io/audit: restricted                  │   │
│   │   pod-security.kubernetes.io/warn: restricted                   │   │
│   │                                                                 │   │
│   │   • Block non-compliant pods                                    │   │
│   │   • Continue auditing and warning                               │   │
│   │   • Full enforcement active                                     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Configuring Pod Security with Namespace Labels

### 3.1 Label Format

```yaml
# Label format
pod-security.kubernetes.io/<MODE>: <PROFILE>
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

| Component | Values |
|-----------|--------|
| **MODE** | `enforce`, `audit`, `warn` |
| **PROFILE** | `privileged`, `baseline`, `restricted` |
| **VERSION** | `latest`, `v1.25`, `v1.26`, etc. |

### 3.2 Example: Baseline Enforcement

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    # Enforce baseline - block violations
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    
    # Audit restricted - log violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    
    # Warn restricted - show warnings
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

### 3.3 Example: Restricted Enforcement

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 3.4 Example: Privileged (System Namespaces)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

### 3.5 Apply Labels to Existing Namespace

```bash
# Add baseline enforcement to existing namespace
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

# Remove label
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce-

# View namespace labels
kubectl get namespace my-app --show-labels
```

---

## 4. Profile Requirements in Detail

### 4.1 Baseline Profile Requirements

The **Baseline** profile blocks known privilege escalations:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BASELINE PROFILE CONTROLS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   MUST NOT have:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   HostProcess (Windows)                                         │   │
│   │   spec.securityContext.windowsOptions.hostProcess               │   │
│   │   spec.containers[*].securityContext.windowsOptions.hostProcess │   │
│   │                                                                 │   │
│   │   Host Namespaces                                               │   │
│   │   spec.hostNetwork: true                                        │   │
│   │   spec.hostPID: true                                            │   │
│   │   spec.hostIPC: true                                            │   │
│   │                                                                 │   │
│   │   Privileged Containers                                         │   │
│   │   spec.containers[*].securityContext.privileged: true           │   │
│   │                                                                 │   │
│   │   Dangerous Capabilities                                        │   │
│   │   capabilities.add: [NET_RAW, ...]  (some allowed)             │   │
│   │   Blocked: ALL, NET_ADMIN, SYS_ADMIN, SYS_PTRACE, etc.         │   │
│   │                                                                 │   │
│   │   HostPath Volumes                                              │   │
│   │   volumes[*].hostPath: {}                                       │   │
│   │                                                                 │   │
│   │   Host Ports                                                    │   │
│   │   containers[*].ports[*].hostPort (restricted range)           │   │
│   │                                                                 │   │
│   │   AppArmor (relaxed in some versions)                          │   │
│   │   Seccomp (Unconfined allowed)                                 │   │
│   │   Sysctls (only safe sysctls allowed)                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Restricted Profile Requirements

The **Restricted** profile includes Baseline plus additional hardening:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   RESTRICTED PROFILE CONTROLS                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Everything from Baseline PLUS:                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Volume Types (limited)                                        │   │
│   │   Only allowed:                                                 │   │
│   │   • configMap, csi, downwardAPI, emptyDir, ephemeral           │   │
│   │   • persistentVolumeClaim, projected, secret                   │   │
│   │   Not allowed: hostPath, nfs, iscsi, etc.                      │   │
│   │                                                                 │   │
│   │   Privilege Escalation                                          │   │
│   │   allowPrivilegeEscalation: false  (REQUIRED)                  │   │
│   │                                                                 │   │
│   │   Running as Non-Root                                           │   │
│   │   runAsNonRoot: true  (REQUIRED)                               │   │
│   │   OR                                                            │   │
│   │   runAsUser: <non-zero>                                        │   │
│   │                                                                 │   │
│   │   Seccomp Profile                                               │   │
│   │   seccompProfile.type: RuntimeDefault or Localhost  (REQUIRED) │   │
│   │   (Unconfined NOT allowed)                                     │   │
│   │                                                                 │   │
│   │   Capabilities                                                  │   │
│   │   capabilities.drop: [ALL]  (REQUIRED)                         │   │
│   │   capabilities.add: only NET_BIND_SERVICE allowed              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Pod That Passes Restricted

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-compliant
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### 4.4 Pod That Fails Restricted

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-violation
spec:
  containers:
  - name: app
    image: nginx
    # VIOLATIONS:
    # 1. No runAsNonRoot: true
    # 2. No allowPrivilegeEscalation: false
    # 3. No capabilities.drop: [ALL]
    # 4. No seccompProfile
```

Error message:
```
Error from server (Forbidden): error when creating "pod.yaml": 
pods "restricted-violation" is forbidden: violates PodSecurity "restricted:latest":
  allowPrivilegeEscalation != false 
  unrestricted capabilities 
  runAsNonRoot != true 
  seccompProfile
```

---

## 5. Dry Run and Testing

### 5.1 Test Pod Against Profile

Before deploying, test if a pod would pass:

```bash
# Check if pod passes restricted profile
kubectl label --dry-run=server --overwrite ns my-namespace \
  pod-security.kubernetes.io/enforce=restricted

# Test specific pod
kubectl apply --dry-run=server -f pod.yaml -n restricted-namespace
```

### 5.2 Using kubectl auth can-i

```bash
# Check if namespace allows non-compliant pods
kubectl auth can-i create pods --namespace=my-app \
  --subresource="" -v=5
```

---

## 6. Exemptions

### 6.1 Cluster-Level Exemptions

Some workloads legitimately need elevated privileges. Configure exemptions in the API server's admission configuration:

```yaml
# /etc/kubernetes/psa-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      # Exempt specific usernames
      usernames: []
      
      # Exempt specific runtime class names
      runtimeClasses: []
      
      # Exempt specific namespaces
      namespaces:
      - kube-system
      - kube-public
      - istio-system
```

### 6.2 Namespace Exemptions

System namespaces typically need privileged access:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    # Full privileges for monitoring agents
    pod-security.kubernetes.io/enforce: privileged
```

---

## 7. Common Patterns

### 7.1 Multi-Tier Namespace Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  MULTI-TIER NAMESPACE STRATEGY                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   TIER 1: System Namespaces (Privileged)                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Namespaces: kube-system, kube-public, monitoring              │   │
│   │  Profile: privileged                                            │   │
│   │  Workloads: CNI, CSI, system agents                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   TIER 2: Shared Services (Baseline)                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Namespaces: logging, ingress, cert-manager                    │   │
│   │  Profile: baseline                                              │   │
│   │  Workloads: Ingress controllers, logging                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   TIER 3: Application Namespaces (Restricted)                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Namespaces: app-*, team-*, prod-*                             │   │
│   │  Profile: restricted                                            │   │
│   │  Workloads: Application pods                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Development vs Production

```yaml
# Development namespace - more permissive
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
---
# Production namespace - maximum security
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 8. Migration from PodSecurityPolicy

### 8.1 PSP Deprecation Timeline

| Version | Status |
|---------|--------|
| K8s 1.21 | PSP deprecated, PSA introduced as beta |
| K8s 1.22 | PSA stable |
| K8s 1.25 | PSP removed |

### 8.2 Migration Steps

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PSP → PSA MIGRATION                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Step 1: Map PSP to PSS Profiles                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Review existing PSPs                                           │   │
│   │  Map each PSP to closest PSS profile:                           │   │
│   │  • Very restrictive PSP → restricted                           │   │
│   │  • Moderate PSP → baseline                                     │   │
│   │  • Permissive PSP → privileged                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Step 2: Label Namespaces (Audit Mode)                                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl label ns <namespace>                                   │   │
│   │    pod-security.kubernetes.io/audit=<profile>                   │   │
│   │                                                                 │   │
│   │  Monitor audit logs for violations                              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Step 3: Add Warn Mode                                                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl label ns <namespace>                                   │   │
│   │    pod-security.kubernetes.io/warn=<profile>                    │   │
│   │                                                                 │   │
│   │  Developers see warnings, fix violations                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Step 4: Enable Enforce Mode                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl label ns <namespace>                                   │   │
│   │    pod-security.kubernetes.io/enforce=<profile>                 │   │
│   │                                                                 │   │
│   │  Violations now blocked                                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Step 5: Remove PSP                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Delete PSP resources                                           │   │
│   │  Remove PSP RBAC bindings                                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Troubleshooting

### 9.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Pod rejected | Violates enforce profile | Fix pod security context |
| Warning but works | Violates warn profile | Fix before enforce is enabled |
| System pods failing | System NS not privileged | Label namespace as privileged |
| Can't run as root | Restricted requires non-root | Set runAsUser and runAsNonRoot |

### 9.2 Debug Commands

```bash
# Check namespace labels
kubectl get ns <namespace> -o yaml | grep pod-security

# Test if pod would be allowed
kubectl apply --dry-run=server -f pod.yaml -n <namespace>

# View PSA violations in events
kubectl get events --field-selector reason=FailedCreate

# Check audit logs (if configured)
cat /var/log/kubernetes/audit.log | grep PodSecurity
```

### 9.3 Understanding Error Messages

```
Error from server (Forbidden): pods "my-pod" is forbidden: 
violates PodSecurity "restricted:v1.25":
  allowPrivilegeEscalation != false 
    (container "app" must set securityContext.allowPrivilegeEscalation=false)
  unrestricted capabilities 
    (container "app" must set securityContext.capabilities.drop=["ALL"])
  runAsNonRoot != true 
    (pod or container "app" must set securityContext.runAsNonRoot=true)
  seccompProfile 
    (pod or container "app" must set securityContext.seccompProfile.type 
     to "RuntimeDefault" or "Localhost")
```

---

## 10. Command Reference

```bash
# Label namespace with enforcement
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce=baseline

# Add all three modes
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Remove label
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce-

# Check namespace labels
kubectl get namespace my-app --show-labels

# Test pod against namespace
kubectl apply --dry-run=server -f pod.yaml -n my-app

# List namespaces with their security profiles
kubectl get namespaces -L pod-security.kubernetes.io/enforce
```

---

## 11. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Label namespace with PSA | 🔴 HIGH | Apply correct labels |
| Understand profiles | 🔴 HIGH | Know privileged/baseline/restricted |
| Fix pod violations | 🟡 MEDIUM | Update securityContext |
| Enforcement modes | 🟡 MEDIUM | enforce, audit, warn |

### Quick Reference for Exam

```yaml
# Namespace labels
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

```bash
# Quick label command
kubectl label ns <namespace> \
  pod-security.kubernetes.io/enforce=baseline
```

### Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Enforce baseline on namespace 'app'"                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl label ns app \                                         │   │
│   │    pod-security.kubernetes.io/enforce=baseline                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Pod rejected - fix to pass restricted"                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Add to pod spec:                                               │   │
│   │    securityContext:                                             │   │
│   │      runAsNonRoot: true                                         │   │
│   │      runAsUser: 1000                                            │   │
│   │      seccompProfile:                                            │   │
│   │        type: RuntimeDefault                                     │   │
│   │    containers:                                                  │   │
│   │    - securityContext:                                           │   │
│   │        allowPrivilegeEscalation: false                          │   │
│   │        capabilities:                                            │   │
│   │          drop: ["ALL"]                                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Allow privileged pods in kube-system"                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl label ns kube-system \                                 │   │
│   │    pod-security.kubernetes.io/enforce=privileged                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    POD SECURITY SUMMARY                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   PROFILES (Security Level):                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  privileged  ──▶  No restrictions (system workloads)           │   │
│   │  baseline    ──▶  Block known exploits (most apps)             │   │
│   │  restricted  ──▶  Maximum hardening (sensitive apps)           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   MODES (Enforcement Level):                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  enforce  ──▶  Block violations                                │   │
│   │  warn     ──▶  Warn on violations (allow)                      │   │
│   │  audit    ──▶  Log violations (silent)                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   LABELS (Configuration):                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  pod-security.kubernetes.io/<mode>: <profile>                   │   │
│   │                                                                 │   │
│   │  Example:                                                       │   │
│   │  pod-security.kubernetes.io/enforce: restricted                │   │
│   │  pod-security.kubernetes.io/warn: restricted                   │   │
│   │  pod-security.kubernetes.io/audit: restricted                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

**Phase 5: Security is COMPLETE! 🎉**

In the next chapter, we'll start **Phase 6: Advanced Topics** with:
- **Resource Management** - Requests, limits, QoS classes
- CPU and memory management
- LimitRanges and ResourceQuotas
- Pod eviction and scheduling

---

**Chapter 20 Complete! 🎉**

You now understand:
- Pod Security Standards (privileged, baseline, restricted)
- Pod Security Admission modes (enforce, audit, warn)
- Configuring PSA with namespace labels
- Profile requirements and comparisons
- Gradual rollout strategy
- Exemptions and migration from PSP
- Troubleshooting common issues
- CKA exam preparation

