# Chapter 28: Cluster Maintenance

## Introduction

Production clusters require regular maintenance: upgrades, backups, node operations, and certificate management. This chapter covers the essential maintenance tasks that are **heavily tested in the CKA exam**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 CLUSTER MAINTENANCE TASKS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
│   │    UPGRADES     │  │    BACKUPS      │  │     NODES       │        │
│   │                 │  │                 │  │                 │        │
│   │  kubeadm        │  │  etcd           │  │  drain          │        │
│   │  kubelet        │  │  Resources      │  │  cordon         │        │
│   │  kubectl        │  │  Restore        │  │  uncordon       │        │
│   │                 │  │                 │  │                 │        │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘        │
│                                                                         │
│   ┌─────────────────┐  ┌─────────────────┐                             │
│   │  CERTIFICATES   │  │   OS UPDATES    │                             │
│   │                 │  │                 │                             │
│   │  Check expiry   │  │  Safe patching  │                             │
│   │  Renewal        │  │  Reboot nodes   │                             │
│   │                 │  │                 │                             │
│   └─────────────────┘  └─────────────────┘                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Cluster Upgrades

### 1.1 Version Skew Policy

Kubernetes components must maintain version compatibility:

| Component | Allowed Skew from API Server |
|-----------|------------------------------|
| kubelet | ±2 minor versions |
| kube-controller-manager | ±1 minor version |
| kube-scheduler | ±1 minor version |
| kubectl | ±1 minor version |

```
Example: If API server is v1.28
├── kubelet: v1.26 - v1.28 (can be up to 2 versions behind)
├── controller-manager: v1.27 - v1.28
├── scheduler: v1.27 - v1.28
└── kubectl: v1.27 - v1.29
```

### 1.2 Upgrade Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    UPGRADE ORDER                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Step 1: Upgrade Control Plane (one node at a time)                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   1. kubeadm upgrade                                            │   │
│   │   2. kubelet & kubectl upgrade                                  │   │
│   │   3. Repeat for other control plane nodes                       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│   Step 2: Upgrade Worker Nodes (one at a time)                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   For each worker node:                                         │   │
│   │   1. Drain the node                                             │   │
│   │   2. Upgrade kubeadm, kubelet, kubectl                          │   │
│   │   3. Uncordon the node                                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   IMPORTANT: Upgrade ONE minor version at a time!                       │
│   v1.27 → v1.28 → v1.29 (NOT v1.27 → v1.29)                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Upgrade Control Plane Node

```bash
# Step 1: Check current version
kubectl get nodes
kubeadm version

# Step 2: Find available versions
apt update
apt-cache madison kubeadm

# Step 3: Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.28.0-00
apt-mark hold kubeadm

# Step 4: Verify upgrade plan
kubeadm upgrade plan

# Step 5: Apply upgrade (first control plane node only)
kubeadm upgrade apply v1.28.0

# For additional control plane nodes:
kubeadm upgrade node

# Step 6: Drain the control plane node
kubectl drain <node-name> --ignore-daemonsets

# Step 7: Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00
apt-mark hold kubelet kubectl

# Step 8: Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# Step 9: Uncordon the node
kubectl uncordon <node-name>
```

### 1.4 Upgrade Worker Node

```bash
# Step 1: Drain the worker node (from control plane)
kubectl drain <worker-node> --ignore-daemonsets --force

# Step 2: SSH to worker node
ssh <worker-node>

# Step 3: Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.28.0-00
apt-mark hold kubeadm

# Step 4: Upgrade node config
kubeadm upgrade node

# Step 5: Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00
apt-mark hold kubelet kubectl

# Step 6: Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# Step 7: Uncordon the node (from control plane)
kubectl uncordon <worker-node>
```

---

## 2. etcd Backup and Restore

### 2.1 Why Backup etcd?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    etcd IS YOUR CLUSTER STATE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   etcd stores EVERYTHING:                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   • All Kubernetes objects (Pods, Deployments, Services, etc.) │   │
│   │   • ConfigMaps and Secrets                                      │   │
│   │   • RBAC configuration                                          │   │
│   │   • Service accounts                                            │   │
│   │   • Persistent volume claims                                    │   │
│   │   • Custom resources                                            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Lose etcd = Lose your ENTIRE cluster configuration!                  │
│                                                                         │
│   ⚠️  THIS IS A HIGH-PRIORITY CKA TOPIC!                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Find etcd Configuration

```bash
# Find etcd pod
kubectl get pods -n kube-system | grep etcd

# Get etcd configuration from pod
kubectl describe pod etcd-<node> -n kube-system

# Or check the static pod manifest
cat /etc/kubernetes/manifests/etcd.yaml

# Key paths you need:
# --cert-file=/etc/kubernetes/pki/etcd/server.crt
# --key-file=/etc/kubernetes/pki/etcd/server.key
# --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
# --listen-client-urls=https://127.0.0.1:2379
```

### 2.3 Backup etcd

```bash
# Set ETCDCTL API version
export ETCDCTL_API=3

# Create backup
etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
etcdctl snapshot status /tmp/etcd-backup.db --write-out=table

# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | abc123   |   123456 |       1234 |     5.0 MB |
# +----------+----------+------------+------------+
```

### 2.4 Restore etcd

```bash
# Stop kube-apiserver (move manifest temporarily)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore the snapshot
etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-backup

# Update etcd manifest to use new data directory
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-backup
# Also update the volume mount path

# Or update hostPath volume:
# volumes:
# - hostPath:
#     path: /var/lib/etcd-backup   # Changed from /var/lib/etcd
#     type: DirectoryOrCreate
#   name: etcd-data

# Start kube-apiserver
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Wait for etcd and apiserver to restart
kubectl get pods -n kube-system
```

### 2.5 etcd Commands Reference

```bash
# Set API version (required!)
export ETCDCTL_API=3

# Common certificate paths (kubeadm clusters)
CACERT=/etc/kubernetes/pki/etcd/ca.crt
CERT=/etc/kubernetes/pki/etcd/server.crt
KEY=/etc/kubernetes/pki/etcd/server.key
ENDPOINT=https://127.0.0.1:2379

# Check etcd health
etcdctl endpoint health \
  --endpoints=$ENDPOINT \
  --cacert=$CACERT \
  --cert=$CERT \
  --key=$KEY

# List etcd members
etcdctl member list \
  --endpoints=$ENDPOINT \
  --cacert=$CACERT \
  --cert=$CERT \
  --key=$KEY

# Create snapshot
etcdctl snapshot save /backup/snapshot.db \
  --endpoints=$ENDPOINT \
  --cacert=$CACERT \
  --cert=$CERT \
  --key=$KEY

# Verify snapshot
etcdctl snapshot status /backup/snapshot.db --write-out=table
```

---

## 3. Node Maintenance

### 3.1 Node Commands

| Command | Description |
|---------|-------------|
| `kubectl cordon <node>` | Mark node unschedulable (no new pods) |
| `kubectl uncordon <node>` | Mark node schedulable again |
| `kubectl drain <node>` | Cordon + evict all pods |

### 3.2 Cordon vs Drain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CORDON vs DRAIN                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CORDON:                                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   kubectl cordon node-1                                         │   │
│   │                                                                 │   │
│   │   • Node marked as SchedulingDisabled                           │   │
│   │   • Existing pods continue running                              │   │
│   │   • No NEW pods scheduled here                                  │   │
│   │                                                                 │   │
│   │   Use case: Prevent new workloads before maintenance           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   DRAIN:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   kubectl drain node-1 --ignore-daemonsets                      │   │
│   │                                                                 │   │
│   │   • Node marked as SchedulingDisabled (cordon)                  │   │
│   │   • All pods EVICTED (gracefully terminated)                    │   │
│   │   • Pods rescheduled to other nodes                             │   │
│   │   • DaemonSets ignored (they stay)                              │   │
│   │                                                                 │   │
│   │   Use case: Before reboot, upgrade, or decommission            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   UNCORDON:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   kubectl uncordon node-1                                       │   │
│   │                                                                 │   │
│   │   • Node marked as schedulable again                           │   │
│   │   • New pods can be scheduled here                             │   │
│   │                                                                 │   │
│   │   Use case: After maintenance complete                         │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Drain Options

```bash
# Basic drain
kubectl drain <node-name>

# Common options:
kubectl drain <node-name> \
  --ignore-daemonsets           # Don't evict DaemonSet pods
  --delete-emptydir-data        # Delete emptyDir volumes
  --force                       # Force evict pods without controller
  --grace-period=30             # Seconds for graceful termination
  --timeout=120s                # How long to wait

# Most common usage:
kubectl drain <node-name> --ignore-daemonsets --force
```

### 3.4 Node Maintenance Workflow

```bash
# 1. Cordon the node (prevent new pods)
kubectl cordon node-1

# 2. Drain the node (evict existing pods)
kubectl drain node-1 --ignore-daemonsets --force

# 3. Perform maintenance (SSH to node)
ssh node-1
# ... do maintenance (OS updates, reboot, etc.)

# 4. Uncordon the node (allow new pods)
kubectl uncordon node-1

# 5. Verify node is ready
kubectl get nodes
```

---

## 4. Certificate Management

### 4.1 Kubernetes Certificates

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 KUBERNETES CERTIFICATES                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Location: /etc/kubernetes/pki/                                        │
│                                                                         │
│   ├── ca.crt, ca.key                    # Cluster CA                   │
│   ├── apiserver.crt, apiserver.key      # API server                   │
│   ├── apiserver-kubelet-client.crt      # API to kubelet               │
│   ├── front-proxy-ca.crt                # Front proxy CA               │
│   └── etcd/                             # etcd certificates            │
│       ├── ca.crt, ca.key                                               │
│       ├── server.crt, server.key                                       │
│       └── peer.crt, peer.key                                           │
│                                                                         │
│   Certificate lifetime: 1 year (by default)                            │
│   CA lifetime: 10 years                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Check Certificate Expiration

```bash
# Check all certificates expiration
kubeadm certs check-expiration

# Output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Dec 15, 2025 10:00 UTC   364d
# apiserver                  Dec 15, 2025 10:00 UTC   364d
# apiserver-etcd-client      Dec 15, 2025 10:00 UTC   364d
# ...

# Check specific certificate with openssl
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 Validity
```

### 4.3 Renew Certificates

```bash
# Renew all certificates
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver

# After renewal, restart control plane components
# (they will auto-restart as they're static pods)

# Verify new expiration
kubeadm certs check-expiration
```

---

## 5. Backing Up Cluster Resources

### 5.1 Backup with kubectl

```bash
# Backup all resources in a namespace
kubectl get all -n <namespace> -o yaml > namespace-backup.yaml

# Backup specific resource types
kubectl get deployments -A -o yaml > deployments-backup.yaml
kubectl get services -A -o yaml > services-backup.yaml
kubectl get configmaps -A -o yaml > configmaps-backup.yaml
kubectl get secrets -A -o yaml > secrets-backup.yaml

# Backup everything
kubectl get all --all-namespaces -o yaml > all-resources.yaml
```

### 5.2 Restore from Backup

```bash
# Restore resources
kubectl apply -f namespace-backup.yaml

# Be careful with:
# - resourceVersion fields (may need to remove)
# - Namespace-specific metadata
# - Secrets may need special handling
```

---

## 6. OS Upgrades

### 6.1 Safe OS Update Process

```bash
# Step 1: Drain the node
kubectl drain <node-name> --ignore-daemonsets --force

# Step 2: SSH to node and perform updates
ssh <node-name>
apt update && apt upgrade -y    # Debian/Ubuntu
# or
yum update -y                    # RHEL/CentOS

# Step 3: Reboot if needed
reboot

# Step 4: Wait for node to come back
kubectl get nodes

# Step 5: Uncordon the node
kubectl uncordon <node-name>

# Step 6: Verify pods are running
kubectl get pods -A -o wide | grep <node-name>
```

---

## 7. Command Reference

```bash
# CLUSTER VERSION
kubectl version
kubeadm version

# UPGRADE COMMANDS
kubeadm upgrade plan                    # Check upgrade path
kubeadm upgrade apply v1.28.0           # Apply upgrade
kubeadm upgrade node                    # Upgrade node config

# NODE MAINTENANCE
kubectl cordon <node>                   # Mark unschedulable
kubectl uncordon <node>                 # Mark schedulable
kubectl drain <node> --ignore-daemonsets --force

# ETCD BACKUP/RESTORE
export ETCDCTL_API=3
etcdctl snapshot save <file>            # Create backup
etcdctl snapshot status <file>          # Verify backup
etcdctl snapshot restore <file>         # Restore backup
etcdctl member list                     # List members
etcdctl endpoint health                 # Check health

# CERTIFICATES
kubeadm certs check-expiration          # Check expiry
kubeadm certs renew all                 # Renew all certs
kubeadm certs renew <cert-name>         # Renew specific

# BACKUP RESOURCES
kubectl get all -A -o yaml > backup.yaml
```

---

## 8. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| etcd backup/restore | 🔴 HIGH | etcdctl snapshot |
| Node drain/cordon | 🔴 HIGH | kubectl drain |
| Cluster upgrade | 🔴 HIGH | kubeadm upgrade |
| Certificate check | 🟡 MEDIUM | kubeadm certs |

### Quick Reference for Exam

```bash
# ETCD BACKUP (memorize this!)
ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# DRAIN NODE
kubectl drain <node> --ignore-daemonsets --force

# UNCORDON NODE
kubectl uncordon <node>
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Backup etcd to /tmp/backup.db"                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  export ETCDCTL_API=3                                           │   │
│   │  etcdctl snapshot save /tmp/backup.db \                         │   │
│   │    --endpoints=https://127.0.0.1:2379 \                         │   │
│   │    --cacert=/etc/kubernetes/pki/etcd/ca.crt \                   │   │
│   │    --cert=/etc/kubernetes/pki/etcd/server.crt \                 │   │
│   │    --key=/etc/kubernetes/pki/etcd/server.key                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Drain node for maintenance"                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl drain <node> --ignore-daemonsets --force               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Upgrade cluster from v1.27 to v1.28"                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  # On control plane:                                            │   │
│   │  apt-get install kubeadm=1.28.0-00                             │   │
│   │  kubeadm upgrade apply v1.28.0                                  │   │
│   │  apt-get install kubelet=1.28.0-00 kubectl=1.28.0-00           │   │
│   │  systemctl daemon-reload && systemctl restart kubelet           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 4: "Check certificate expiration"                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubeadm certs check-expiration                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                CLUSTER MAINTENANCE SUMMARY                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   UPGRADES:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  1. Upgrade control plane first                                 │   │
│   │  2. Upgrade workers one at a time                               │   │
│   │  3. One minor version at a time                                 │   │
│   │  4. kubeadm upgrade → kubelet → kubectl                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ETCD BACKUP (CRITICAL!):                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  ETCDCTL_API=3 etcdctl snapshot save <file>                    │   │
│   │    --endpoints --cacert --cert --key                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   NODE MAINTENANCE:                                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl drain → maintenance → kubectl uncordon                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   CERTIFICATES:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubeadm certs check-expiration                                 │   │
│   │  kubeadm certs renew all                                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the final chapter, we'll cover:
- **CKA Exam Preparation** - Tips, strategies, and resources
- Exam format and domains
- Practice exercises
- Time management
- Recommended resources

---

**Chapter 28 Complete! 🎉**

You now understand:
- Cluster upgrade process with kubeadm
- etcd backup and restore (critical for CKA!)
- Node maintenance (drain, cordon, uncordon)
- Certificate management
- OS upgrade procedures
- Resource backup strategies

