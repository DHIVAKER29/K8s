# Chapter 27: Troubleshooting

## Introduction

Things break. Pods crash, nodes fail, networks disconnect. The ability to systematically troubleshoot Kubernetes issues is **critical for CKA** and real-world operations. This chapter covers common problems and how to solve them.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 TROUBLESHOOTING METHODOLOGY                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1. IDENTIFY THE SYMPTOM                                               │
│      │                                                                  │
│      │   "Pod is not running"                                          │
│      │   "Service is not accessible"                                   │
│      │   "Node is NotReady"                                            │
│      │                                                                  │
│      ▼                                                                  │
│   2. GATHER INFORMATION                                                 │
│      │                                                                  │
│      │   kubectl get <resource>                                        │
│      │   kubectl describe <resource>                                   │
│      │   kubectl logs <pod>                                            │
│      │   kubectl get events                                            │
│      │                                                                  │
│      ▼                                                                  │
│   3. ANALYZE THE DATA                                                   │
│      │                                                                  │
│      │   Look for error messages                                       │
│      │   Check events and status                                       │
│      │   Identify the root cause                                       │
│      │                                                                  │
│      ▼                                                                  │
│   4. IMPLEMENT THE FIX                                                  │
│      │                                                                  │
│      │   Apply the solution                                            │
│      │   Verify the fix worked                                         │
│      │                                                                  │
│      ▼                                                                  │
│   5. VERIFY & DOCUMENT                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Pod Troubleshooting

### 1.1 Pod Status Reference

| Status | Meaning | Common Causes |
|--------|---------|---------------|
| `Pending` | Pod accepted but not scheduled | No resources, no nodes match |
| `Running` | Pod is running | Normal state |
| `Succeeded` | All containers exited with 0 | Job completed |
| `Failed` | All containers stopped, one failed | App error |
| `Unknown` | Can't get pod state | Node communication lost |
| `CrashLoopBackOff` | Container keeps crashing | App crashes on startup |
| `ImagePullBackOff` | Can't pull image | Wrong image, no auth |
| `ErrImagePull` | Error pulling image | Network, auth, wrong name |
| `CreateContainerConfigError` | Config error | Bad ConfigMap/Secret ref |
| `ContainerCreating` | Container starting | Pulling image, mounting volumes |

### 1.2 Pending Pod

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PENDING POD                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Pod Status: Pending                                                   │
│                                                                         │
│   Common Causes:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  1. Insufficient Resources                                      │   │
│   │     • Not enough CPU/memory on any node                        │   │
│   │     • Check: kubectl describe node | grep -A5 Allocated        │   │
│   │                                                                 │   │
│   │  2. Node Selector/Affinity Not Met                              │   │
│   │     • No nodes match nodeSelector or affinity rules            │   │
│   │     • Check: kubectl get nodes --show-labels                   │   │
│   │                                                                 │   │
│   │  3. Taints Not Tolerated                                        │   │
│   │     • Nodes have taints pod doesn't tolerate                   │   │
│   │     • Check: kubectl describe node | grep Taints               │   │
│   │                                                                 │   │
│   │  4. PVC Not Bound                                               │   │
│   │     • PersistentVolumeClaim waiting for PV                     │   │
│   │     • Check: kubectl get pvc                                   │   │
│   │                                                                 │   │
│   │  5. ResourceQuota Exceeded                                      │   │
│   │     • Namespace quota reached                                  │   │
│   │     • Check: kubectl describe resourcequota                    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Debug Commands:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe pod <pod>          # Check Events section    │   │
│   │  kubectl get events --sort-by='.lastTimestamp'                 │   │
│   │  kubectl describe nodes | grep -A5 "Allocated resources"       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 CrashLoopBackOff

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CRASHLOOPBACKOFF                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   What it means: Container starts, crashes, Kubernetes restarts it,     │
│   it crashes again, repeat with exponential backoff                     │
│                                                                         │
│   Timeline:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Start → Crash → Wait 10s → Start → Crash → Wait 20s → ...    │   │
│   │                                                                 │   │
│   │  Backoff: 10s, 20s, 40s, 80s, 160s, 300s (max 5 min)          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Common Causes:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  1. Application Error                                           │   │
│   │     • App crashes on startup                                   │   │
│   │     • Check: kubectl logs <pod> --previous                     │   │
│   │                                                                 │   │
│   │  2. Wrong Command/Args                                          │   │
│   │     • Incorrect entrypoint or arguments                        │   │
│   │     • Check: kubectl describe pod <pod>                        │   │
│   │                                                                 │   │
│   │  3. Missing Dependencies                                        │   │
│   │     • Database not available, config missing                   │   │
│   │     • Check: kubectl logs <pod> --previous                     │   │
│   │                                                                 │   │
│   │  4. Liveness Probe Failing                                      │   │
│   │     • Probe kills container before it's ready                  │   │
│   │     • Check: kubectl describe pod | grep Liveness              │   │
│   │                                                                 │   │
│   │  5. OOM Killed                                                  │   │
│   │     • Container exceeded memory limit                          │   │
│   │     • Check: kubectl describe pod | grep OOMKilled             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Debug Commands:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl logs <pod> --previous       # Previous container logs │   │
│   │  kubectl describe pod <pod>          # Check exit code, events │   │
│   │  kubectl get pod <pod> -o yaml | grep -A10 lastState          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.4 ImagePullBackOff / ErrImagePull

```bash
# Common causes:
# 1. Wrong image name
# 2. Image doesn't exist
# 3. Private registry without credentials
# 4. Network issues

# Debug:
kubectl describe pod <pod>   # Look at Events

# Events you might see:
# Failed to pull image "nginx:wrongtag": rpc error: ... not found
# Failed to pull image: unauthorized: authentication required

# Solutions:
# 1. Fix image name
# 2. Create imagePullSecret for private registry
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<password>
```

### 1.5 CreateContainerConfigError

```bash
# Common causes:
# - ConfigMap doesn't exist
# - Secret doesn't exist
# - Key not found in ConfigMap/Secret

# Debug:
kubectl describe pod <pod>

# Events:
# Error: configmap "app-config" not found
# Error: secret "app-secret" not found

# Solution: Create the missing ConfigMap/Secret
kubectl get configmaps
kubectl get secrets
```

### 1.6 Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0 | Success (container completed normally) |
| 1 | Application error |
| 126 | Command cannot execute (permission) |
| 127 | Command not found |
| 128 | Invalid exit argument |
| 137 | SIGKILL (128 + 9) - OOM or kubectl delete |
| 139 | SIGSEGV (128 + 11) - Segmentation fault |
| 143 | SIGTERM (128 + 15) - Graceful termination |

```bash
# Check exit code
kubectl describe pod <pod> | grep -A5 "Last State"

# Output:
# Last State:     Terminated
#   Reason:       OOMKilled     # or Error
#   Exit Code:    137           # OOM killed
```

---

## 2. Node Troubleshooting

### 2.1 Node Status

```bash
# Check node status
kubectl get nodes

# Output:
# NAME     STATUS     ROLES    AGE   VERSION
# node-1   Ready      master   10d   v1.28.0
# node-2   NotReady   <none>   10d   v1.28.0

# Get more details
kubectl describe node <node-name>
```

### 2.2 NotReady Node

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     NOTREADY NODE                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Common Causes:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  1. Kubelet Not Running                                         │   │
│   │     • ssh to node: systemctl status kubelet                    │   │
│   │     • Fix: systemctl start kubelet                             │   │
│   │                                                                 │   │
│   │  2. Container Runtime Not Running                               │   │
│   │     • ssh to node: systemctl status containerd                 │   │
│   │     • Fix: systemctl start containerd                          │   │
│   │                                                                 │   │
│   │  3. Network Issues                                              │   │
│   │     • Node can't reach API server                              │   │
│   │     • Check: ping <api-server-ip>                              │   │
│   │                                                                 │   │
│   │  4. Certificate Issues                                          │   │
│   │     • Kubelet certificates expired                             │   │
│   │     • Check: openssl x509 -in /var/lib/kubelet/pki/...         │   │
│   │                                                                 │   │
│   │  5. Resource Pressure                                           │   │
│   │     • DiskPressure, MemoryPressure, PIDPressure                │   │
│   │     • Check: kubectl describe node | grep Conditions           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Debug Commands (on the node):                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  systemctl status kubelet                                      │   │
│   │  journalctl -u kubelet -f                                      │   │
│   │  systemctl status containerd                                   │   │
│   │  crictl ps                                                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Node Conditions

```bash
kubectl describe node <node> | grep -A10 Conditions

# Conditions:
#   Type             Status  Reason
#   ----             ------  ------
#   MemoryPressure   False   KubeletHasSufficientMemory
#   DiskPressure     False   KubeletHasNoDiskPressure
#   PIDPressure      False   KubeletHasSufficientPID
#   Ready            True    KubeletReady
```

| Condition | Meaning |
|-----------|---------|
| `Ready` | Node can accept pods |
| `MemoryPressure` | Low memory |
| `DiskPressure` | Low disk space |
| `PIDPressure` | Too many processes |
| `NetworkUnavailable` | Network not configured |

### 2.4 Node Debug Commands

```bash
# SSH to node first
ssh <node>

# Check kubelet
systemctl status kubelet
journalctl -u kubelet --since "5 minutes ago"

# Check container runtime
systemctl status containerd
crictl ps                     # List running containers
crictl pods                   # List pods

# Check resources
df -h                         # Disk space
free -m                       # Memory
top                           # CPU/processes

# Check network
ip addr                       # Network interfaces
ping <api-server>             # API server connectivity
```

---

## 3. Service Troubleshooting

### 3.1 Service Not Working

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  SERVICE TROUBLESHOOTING                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Checklist:                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  1. Is the Service created?                                     │   │
│   │     kubectl get svc <service-name>                             │   │
│   │                                                                 │   │
│   │  2. Does Service selector match Pod labels?                     │   │
│   │     kubectl get svc <service> -o yaml | grep selector          │   │
│   │     kubectl get pods --show-labels                             │   │
│   │                                                                 │   │
│   │  3. Are there Endpoints?                                        │   │
│   │     kubectl get endpoints <service-name>                       │   │
│   │     (If empty, selector doesn't match any pods)                │   │
│   │                                                                 │   │
│   │  4. Are Pods Ready?                                             │   │
│   │     kubectl get pods                                           │   │
│   │     (Only Ready pods are in Endpoints)                         │   │
│   │                                                                 │   │
│   │  5. Is target port correct?                                     │   │
│   │     Service targetPort must match container port               │   │
│   │                                                                 │   │
│   │  6. Is kube-proxy running?                                      │   │
│   │     kubectl get pods -n kube-system -l k8s-app=kube-proxy     │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Debug Commands

```bash
# Check service
kubectl get svc <service>
kubectl describe svc <service>

# Check endpoints (critical!)
kubectl get endpoints <service>

# If endpoints empty:
# 1. Check selector matches pod labels
kubectl get svc <service> -o yaml | grep -A5 selector
kubectl get pods --show-labels

# Test connectivity from within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- <service>:<port>

# Check DNS
kubectl run test --image=busybox --rm -it -- nslookup <service>
```

---

## 4. Networking Troubleshooting

### 4.1 DNS Issues

```bash
# Test DNS resolution
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes

# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check DNS config in pod
kubectl exec <pod> -- cat /etc/resolv.conf
```

### 4.2 Pod-to-Pod Connectivity

```bash
# Get pod IPs
kubectl get pods -o wide

# Test connectivity between pods
kubectl exec <pod1> -- ping <pod2-ip>
kubectl exec <pod1> -- wget -qO- <pod2-ip>:<port>

# If failing, check:
# 1. Network policy blocking traffic?
kubectl get networkpolicy

# 2. CNI plugin working?
kubectl get pods -n kube-system   # Look for calico, flannel, etc.
```

---

## 5. Control Plane Troubleshooting

### 5.1 API Server Issues

```bash
# Check API server pod (kubeadm clusters)
kubectl get pods -n kube-system | grep apiserver

# Check API server logs
kubectl logs -n kube-system kube-apiserver-<node>

# If kubectl not working:
# SSH to master node
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check systemd service
systemctl status kubelet   # kubelet runs static pods
journalctl -u kubelet
```

### 5.2 etcd Issues

```bash
# Check etcd pod
kubectl get pods -n kube-system | grep etcd

# Check etcd logs
kubectl logs -n kube-system etcd-<node>

# etcd health (on master node)
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 5.3 Scheduler/Controller Manager

```bash
# Check scheduler
kubectl get pods -n kube-system | grep scheduler
kubectl logs -n kube-system kube-scheduler-<node>

# Check controller manager
kubectl get pods -n kube-system | grep controller
kubectl logs -n kube-system kube-controller-manager-<node>
```

---

## 6. Debug Containers

### 6.1 Ephemeral Debug Containers

```bash
# Add debug container to running pod (K8s 1.23+)
kubectl debug <pod> -it --image=busybox --target=<container>

# Debug with different image
kubectl debug <pod> -it --image=nicolaka/netshoot

# Create copy of pod with debug container
kubectl debug <pod> -it --image=busybox --copy-to=debug-pod
```

### 6.2 Debug Node

```bash
# Create privileged pod on node for debugging
kubectl debug node/<node-name> -it --image=ubuntu

# This gives you access to node filesystem at /host
ls /host/var/log
cat /host/etc/kubernetes/kubelet.conf
```

---

## 7. Common Problems Quick Reference

### 7.1 Problem → Solution Table

| Problem | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| Pod Pending (no node) | No resources | Check node capacity, add nodes |
| Pod Pending (PVC) | PVC not bound | Create PV or fix StorageClass |
| ImagePullBackOff | Wrong image/no auth | Fix image name, add imagePullSecret |
| CrashLoopBackOff | App crashing | Check logs with `--previous` |
| OOMKilled | Memory limit too low | Increase memory limit |
| Node NotReady | Kubelet down | SSH, `systemctl start kubelet` |
| Service no endpoints | Selector mismatch | Fix labels to match selector |
| DNS not working | CoreDNS down | Restart CoreDNS pods |
| Can't schedule | Taints/tolerations | Add toleration or remove taint |

### 7.2 Quick Debug Flowchart

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    QUICK DEBUG FLOWCHART                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Pod not running?                                                      │
│        │                                                                │
│        ├── Pending? ──▶ kubectl describe pod (check Events)            │
│        │                ├── No resources? Add nodes or reduce requests │
│        │                ├── PVC pending? Check PV/StorageClass         │
│        │                └── Taints? Add toleration                     │
│        │                                                                │
│        ├── CrashLoop? ──▶ kubectl logs --previous                      │
│        │                  ├── App error? Fix app                       │
│        │                  ├── OOMKilled? Increase memory               │
│        │                  └── Missing config? Create ConfigMap/Secret  │
│        │                                                                │
│        └── ImagePull? ──▶ kubectl describe pod (check Events)          │
│                          ├── Wrong name? Fix image name                │
│                          └── Private? Create imagePullSecret           │
│                                                                         │
│   Service not working?                                                  │
│        │                                                                │
│        └── Check endpoints ──▶ kubectl get endpoints <svc>             │
│                                ├── Empty? Fix selector/labels          │
│                                └── Has IPs? Check targetPort           │
│                                                                         │
│   Node NotReady?                                                        │
│        │                                                                │
│        └── SSH to node ──▶ systemctl status kubelet                    │
│                            └── Not running? systemctl start kubelet    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Command Reference

```bash
# POD DEBUGGING
kubectl get pods                           # List pods
kubectl get pods -o wide                   # With node info
kubectl describe pod <pod>                 # Detailed info + events
kubectl logs <pod>                         # Container logs
kubectl logs <pod> --previous              # Previous container
kubectl logs <pod> -c <container>          # Specific container
kubectl exec -it <pod> -- /bin/sh          # Shell into container

# NODE DEBUGGING
kubectl get nodes                          # List nodes
kubectl describe node <node>               # Detailed info
kubectl top nodes                          # Resource usage
kubectl debug node/<node> -it --image=ubuntu  # Debug node

# SERVICE DEBUGGING
kubectl get svc                            # List services
kubectl describe svc <service>             # Service details
kubectl get endpoints <service>            # Check endpoints

# EVENTS
kubectl get events                         # All events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod>

# CLUSTER COMPONENTS
kubectl get pods -n kube-system            # System pods
kubectl logs -n kube-system <pod>          # Component logs
```

---

## 9. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Pod troubleshooting | 🔴 HIGH | describe, logs, events |
| Node troubleshooting | 🔴 HIGH | kubelet, journalctl |
| Service debugging | 🔴 HIGH | endpoints, selectors |
| Application logs | 🔴 HIGH | kubectl logs |

### Quick Reference for Exam

```bash
# The 4 most important commands:
kubectl describe pod <pod>       # Events and status
kubectl logs <pod> --previous    # Why did it crash?
kubectl get events               # Cluster events
kubectl get endpoints <svc>      # Service working?
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Pod is not starting, fix it"                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe pod <pod>    # Find the issue                 │   │
│   │  kubectl logs <pod> --previous # If CrashLoopBackOff           │   │
│   │  kubectl edit pod <pod>        # Or delete and recreate        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Node is NotReady, fix it"                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  ssh <node>                                                     │   │
│   │  systemctl status kubelet                                       │   │
│   │  systemctl start kubelet                                        │   │
│   │  journalctl -u kubelet         # If still failing              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Service is not accessible"                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl get endpoints <svc>   # Check if pods in endpoints    │   │
│   │  kubectl get svc <svc> -o yaml # Check selector                │   │
│   │  kubectl get pods --show-labels # Check pod labels             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 4: "Application logs needed"                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl logs <pod> > /path/to/output.log                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   TROUBLESHOOTING SUMMARY                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   POD ISSUES:                                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe pod → Check Events                           │   │
│   │  kubectl logs --previous → See crash reason                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   NODE ISSUES:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  SSH to node                                                   │   │
│   │  systemctl status/start kubelet                                │   │
│   │  journalctl -u kubelet                                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   SERVICE ISSUES:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl get endpoints → Empty = selector problem              │   │
│   │  Match service selector to pod labels                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   REMEMBER:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  1. describe → logs → events                                   │   │
│   │  2. Check the Events section first!                            │   │
│   │  3. Empty endpoints = selector mismatch                        │   │
│   │  4. Node NotReady = SSH and check kubelet                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Cluster Maintenance** - Upgrades, backups, node operations
- Cluster upgrades with kubeadm
- etcd backup and restore
- Node drain and maintenance

---

**Chapter 27 Complete! 🎉**

You now understand:
- Pod troubleshooting (Pending, CrashLoopBackOff, ImagePullBackOff)
- Node troubleshooting (NotReady, kubelet issues)
- Service troubleshooting (endpoints, selectors)
- Networking issues (DNS, connectivity)
- Control plane troubleshooting
- Debug containers
- Systematic debugging approach

