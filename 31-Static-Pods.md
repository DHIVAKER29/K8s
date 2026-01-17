# Chapter 31: Static Pods

## Introduction

Most pods are created through the API server (Deployments, ReplicaSets, Jobs). But **Static Pods** are different - they're managed directly by the kubelet on a specific node, without the API server's involvement. This is how Kubernetes control plane components run!

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  STATIC PODS vs REGULAR PODS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Regular Pods:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   User ──▶ kubectl apply ──▶ API Server ──▶ etcd               │   │
│   │                                    │                            │   │
│   │                                    ▼                            │   │
│   │                              Scheduler                          │   │
│   │                                    │                            │   │
│   │                                    ▼                            │   │
│   │                           Kubelet ──▶ Creates Pod              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Static Pods:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   YAML file in /etc/kubernetes/manifests/                       │   │
│   │        │                                                        │   │
│   │        ▼                                                        │   │
│   │   Kubelet watches directory ──▶ Creates Pod directly           │   │
│   │        │                                                        │   │
│   │        ▼                                                        │   │
│   │   API Server shows "mirror pod" (read-only)                    │   │
│   │                                                                 │   │
│   │   NO etcd, NO scheduler, NO controllers!                       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. What are Static Pods?

### 1.1 Key Characteristics

| Feature | Static Pod | Regular Pod |
|---------|------------|-------------|
| Created by | Kubelet directly | API Server |
| Stored in | Local filesystem | etcd |
| Scheduled by | No scheduler (runs on specific node) | Scheduler |
| Managed by | Kubelet | Controllers |
| Visible in API | Yes (mirror pod, read-only) | Yes (full control) |
| Can be deleted via kubectl | No (recreated immediately) | Yes |

### 1.2 Use Cases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STATIC POD USE CASES                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1. Control Plane Components (kubeadm clusters)                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  /etc/kubernetes/manifests/                                     │   │
│   │  ├── kube-apiserver.yaml                                        │   │
│   │  ├── kube-controller-manager.yaml                               │   │
│   │  ├── kube-scheduler.yaml                                        │   │
│   │  └── etcd.yaml                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   2. Node-Critical Services                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Services that must run even if API server is down            │   │
│   │  • Monitoring agents                                            │   │
│   │  • Log collectors                                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   3. Bootstrap Scenarios                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Chicken-and-egg: API server needs to run before API works   │   │
│   │  • Static pods solve this!                                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Static Pod Location

### 2.1 Finding the Static Pod Path

```bash
# Check kubelet configuration for static pod path
# Method 1: Check kubelet service
systemctl status kubelet
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Method 2: Check kubelet process
ps aux | grep kubelet | grep -- --pod-manifest-path

# Default locations:
# /etc/kubernetes/manifests/     (kubeadm default)
# /etc/kubelet.d/                (some distributions)
```

### 2.2 Kubelet Configuration

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
staticPodPath: /etc/kubernetes/manifests   # Static pods here
```

---

## 3. Creating Static Pods

### 3.1 Create a Static Pod

```bash
# SSH to the node
ssh <node-name>

# Create manifest in static pod directory
cat <<EOF > /etc/kubernetes/manifests/static-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-nginx
  labels:
    app: static-web
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
EOF

# Kubelet automatically creates the pod!
# No kubectl apply needed!
```

### 3.2 Verify Static Pod

```bash
# On the node
crictl pods | grep static-nginx

# From master (shows mirror pod)
kubectl get pods -A | grep static-nginx

# Output includes node name suffix:
# static-nginx-<node-name>
```

---

## 4. Mirror Pods

### 4.1 What is a Mirror Pod?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      MIRROR PODS                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   When kubelet creates a static pod, it also creates a "mirror pod"   │
│   in the API server for visibility.                                    │
│                                                                         │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │                        NODE                                   │     │
│   │                                                               │     │
│   │   /etc/kubernetes/manifests/static-nginx.yaml                │     │
│   │                    │                                          │     │
│   │                    ▼                                          │     │
│   │              ┌──────────┐                                     │     │
│   │              │  Kubelet │                                     │     │
│   │              └────┬─────┘                                     │     │
│   │                   │                                           │     │
│   │         ┌─────────┴─────────┐                                │     │
│   │         │                   │                                │     │
│   │         ▼                   ▼                                │     │
│   │   ┌──────────┐      ┌─────────────┐                         │     │
│   │   │   Pod    │      │ Mirror Pod  │ ──▶ API Server          │     │
│   │   │ (actual) │      │ (read-only) │                         │     │
│   │   └──────────┘      └─────────────┘                         │     │
│   │                                                               │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│   Mirror pod naming: <static-pod-name>-<node-name>                     │
│   Example: static-nginx-worker-1                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Mirror Pod Properties

```bash
# View mirror pod
kubectl get pod static-nginx-node01 -o yaml

# You'll see:
# metadata:
#   annotations:
#     kubernetes.io/config.mirror: <hash>   # Marks it as mirror
#   ownerReferences:
#     - kind: Node                           # Owned by node
```

---

## 5. Managing Static Pods

### 5.1 Modifying Static Pods

```bash
# To update a static pod:
# 1. SSH to the node
ssh <node-name>

# 2. Edit the manifest file
vi /etc/kubernetes/manifests/static-nginx.yaml

# 3. Kubelet automatically recreates the pod
# No kubectl apply needed!

# To verify
kubectl get pod static-nginx-<node-name> -o yaml
```

### 5.2 Deleting Static Pods

```bash
# Method 1: Remove manifest file (correct way)
ssh <node-name>
rm /etc/kubernetes/manifests/static-nginx.yaml

# Method 2: kubectl delete (won't work permanently!)
kubectl delete pod static-nginx-node01
# Pod will be recreated immediately by kubelet!
```

### 5.3 Why kubectl delete Doesn't Work

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DELETING STATIC PODS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   kubectl delete pod static-nginx-node01                                │
│        │                                                                │
│        ▼                                                                │
│   API Server deletes mirror pod                                         │
│        │                                                                │
│        ▼                                                                │
│   Kubelet sees manifest still exists                                    │
│        │                                                                │
│        ▼                                                                │
│   Kubelet recreates pod + mirror pod                                    │
│        │                                                                │
│        ▼                                                                │
│   Pod is back! (within seconds)                                         │
│                                                                         │
│   To permanently delete: Remove the manifest file from node!           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Control Plane as Static Pods

### 6.1 kubeadm Control Plane

```bash
# On a kubeadm master node:
ls /etc/kubernetes/manifests/

# Output:
# etcd.yaml
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml

# These are all static pods!
kubectl get pods -n kube-system

# Output includes:
# etcd-master                          1/1     Running
# kube-apiserver-master                1/1     Running
# kube-controller-manager-master       1/1     Running
# kube-scheduler-master                1/1     Running
```

### 6.2 Modifying Control Plane Components

```bash
# Example: Change API server configuration
# 1. Edit the static pod manifest
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# 2. Add/modify flags
# spec:
#   containers:
#   - command:
#     - kube-apiserver
#     - --audit-log-path=/var/log/audit.log  # Add this

# 3. Save file - kubelet auto-restarts the pod

# 4. Verify
kubectl get pods -n kube-system | grep apiserver
```

---

## 7. Identifying Static Pods

### 7.1 How to Identify

```bash
# Method 1: Check pod name (has node suffix)
kubectl get pods -A
# etcd-master                    (static - has node suffix)
# kube-apiserver-master          (static - has node suffix)
# nginx-deployment-abc123        (regular - random suffix)

# Method 2: Check ownerReferences
kubectl get pod <pod-name> -o yaml | grep -A5 ownerReferences
# Static pod: ownerReferences.kind = Node
# Regular pod: ownerReferences.kind = ReplicaSet (or other controller)

# Method 3: Check annotations
kubectl get pod <pod-name> -o yaml | grep config.mirror
# Static pods have kubernetes.io/config.mirror annotation
```

### 7.2 Quick Identification

| Indicator | Static Pod | Regular Pod |
|-----------|------------|-------------|
| Name suffix | `-<node-name>` | Random hash |
| Owner | Node | ReplicaSet/Job/etc. |
| Has mirror annotation | Yes | No |
| Location | kube-system (usually) | Any namespace |

---

## 8. CKA Exam Tips

### 8.1 Common Exam Tasks

```bash
# Task: "Create a static pod named static-busybox on node01"
# Solution:
ssh node01
cat <<EOF > /etc/kubernetes/manifests/static-busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-busybox
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sleep', '3600']
EOF
exit

# Verify:
kubectl get pods | grep static-busybox
```

### 8.2 Finding Static Pod Path

```bash
# Quick method for exam:
ps aux | grep kubelet | grep -- --pod-manifest-path

# Or check config:
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Default: /etc/kubernetes/manifests/
```

### 8.3 Common Scenarios

| Scenario | Solution |
|----------|----------|
| Create static pod | Create YAML in `/etc/kubernetes/manifests/` |
| Delete static pod | Remove YAML from `/etc/kubernetes/manifests/` |
| Modify static pod | Edit YAML file (auto-restarts) |
| Find static pod path | Check kubelet config |
| Identify static pod | Look for node name suffix |

---

## 9. Command Reference

```bash
# Find static pod directory
cat /var/lib/kubelet/config.yaml | grep staticPodPath
ps aux | grep kubelet | grep manifest

# Create static pod
cat <<EOF > /etc/kubernetes/manifests/my-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Delete static pod
rm /etc/kubernetes/manifests/my-pod.yaml

# List static pods (look for node suffix)
kubectl get pods -A | grep -E "master|node"

# Check if pod is static
kubectl get pod <name> -o yaml | grep config.mirror
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STATIC PODS SUMMARY                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   WHAT: Pods managed directly by kubelet, not API server               │
│                                                                         │
│   WHERE: /etc/kubernetes/manifests/ (default)                          │
│                                                                         │
│   WHY:                                                                  │
│   • Control plane components (etcd, apiserver, scheduler, controller)  │
│   • Pods that must run without API server                              │
│                                                                         │
│   HOW TO CREATE:                                                        │
│   • Put YAML manifest in static pod directory                          │
│   • Kubelet automatically creates the pod                              │
│                                                                         │
│   HOW TO DELETE:                                                        │
│   • Remove manifest file from directory                                │
│   • kubectl delete won't work (pod is recreated)                       │
│                                                                         │
│   IDENTIFICATION:                                                       │
│   • Pod name ends with -<node-name>                                    │
│   • Has kubernetes.io/config.mirror annotation                         │
│   • Owner is Node, not ReplicaSet                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**Chapter 31 Complete!** ✅

