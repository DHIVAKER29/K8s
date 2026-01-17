# Chapter 29: CKA Exam Preparation

## 🎉 Congratulations!

You've completed the Kubernetes learning journey! This final chapter prepares you specifically for the **Certified Kubernetes Administrator (CKA)** exam.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CKA CERTIFICATION                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │        ╔═══════════════════════════════════════════╗            │   │
│   │        ║                                           ║            │   │
│   │        ║   CERTIFIED KUBERNETES ADMINISTRATOR     ║            │   │
│   │        ║                                           ║            │   │
│   │        ║              [YOUR NAME]                  ║            │   │
│   │        ║                                           ║            │   │
│   │        ║        The Linux Foundation               ║            │   │
│   │        ║              & CNCF                       ║            │   │
│   │        ║                                           ║            │   │
│   │        ╚═══════════════════════════════════════════╝            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Format: Performance-based (hands-on)                                  │
│   Duration: 2 hours                                                     │
│   Questions: 15-20 tasks                                                │
│   Passing Score: 66%                                                    │
│   Retake: 1 free retake included                                        │
│   Validity: 3 years                                                     │
│   Cost: $395 USD                                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Exam Format

### 1.1 Exam Details

| Aspect | Details |
|--------|---------|
| **Type** | Performance-based (hands-on) |
| **Duration** | 2 hours |
| **Questions** | 15-20 practical tasks |
| **Passing Score** | 66% |
| **Environment** | Real Kubernetes clusters |
| **Resources** | kubernetes.io docs allowed |
| **Proctored** | Yes, online |
| **Retake** | 1 free retake included |
| **Validity** | 3 years |

### 1.2 Exam Environment

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EXAM ENVIRONMENT                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   You get:                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   • Remote desktop with terminal                               │   │
│   │   • Multiple Kubernetes clusters                               │   │
│   │   • kubectl, vim, nano editors                                 │   │
│   │   • kubernetes.io documentation (1 tab allowed)                │   │
│   │   • Task descriptions with requirements                        │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   You must:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   • Switch between clusters per question                       │   │
│   │   • Read context carefully (namespace, cluster)                │   │
│   │   • Complete tasks using command line                          │   │
│   │   • Verify your work before moving on                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Exam Domains (2024)

### 2.1 Domain Weights

| Domain | Weight | Topics |
|--------|--------|--------|
| **Storage** | 10% | PV, PVC, StorageClass, volumes |
| **Troubleshooting** | 30% | Pod/node/network issues, logs |
| **Workloads & Scheduling** | 15% | Deployments, scheduling, scaling |
| **Cluster Architecture** | 25% | RBAC, upgrades, etcd, HA |
| **Services & Networking** | 20% | Services, Ingress, DNS, NetworkPolicy |

### 2.2 Domain Breakdown

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOMAIN WEIGHTS                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Troubleshooting (30%)          ████████████████████████████████       │
│   Most important! Debug pods, nodes, clusters                          │
│                                                                         │
│   Cluster Architecture (25%)     ██████████████████████████             │
│   RBAC, upgrades, etcd backup/restore, HA                              │
│                                                                         │
│   Services & Networking (20%)    █████████████████████                  │
│   Services, Ingress, DNS, NetworkPolicy                                │
│                                                                         │
│   Workloads & Scheduling (15%)   ███████████████                        │
│   Deployments, DaemonSets, scheduling                                  │
│                                                                         │
│   Storage (10%)                  ██████████                             │
│   PV, PVC, StorageClass                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Essential kubectl Shortcuts

### 3.1 Aliases (Set at Start of Exam)

```bash
# These may already be set, but verify:
alias k=kubectl

# Auto-completion (should be enabled)
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Useful aliases to add
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kaf='kubectl apply -f'
export do='--dry-run=client -o yaml'
```

### 3.2 Time-Saving Commands

```bash
# Generate YAML quickly
kubectl run nginx --image=nginx $do > pod.yaml
kubectl create deployment web --image=nginx $do > deploy.yaml
kubectl expose deployment web --port=80 $do > svc.yaml

# Quick pod with command
kubectl run busybox --image=busybox --rm -it -- sh

# Set namespace for session
kubectl config set-context --current --namespace=<namespace>

# Quick resource creation
kubectl create configmap my-cm --from-literal=key=value
kubectl create secret generic my-secret --from-literal=password=secret
kubectl create serviceaccount my-sa
```

### 3.3 vim Settings

```bash
# Add to ~/.vimrc for YAML editing
set expandtab
set tabstop=2
set shiftwidth=2
set autoindent

# Quick command in vim:
:set paste     # Before pasting YAML
:set nopaste   # After pasting
```

---

## 4. High-Priority Topics

### 4.1 Must-Know Commands

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MUST-KNOW COMMANDS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   POD OPERATIONS:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl run nginx --image=nginx                                │   │
│   │  kubectl get pods -o wide                                       │   │
│   │  kubectl describe pod <pod>                                     │   │
│   │  kubectl logs <pod> [--previous] [-c container]                │   │
│   │  kubectl exec -it <pod> -- /bin/sh                             │   │
│   │  kubectl delete pod <pod> --force --grace-period=0             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   DEPLOYMENTS:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl create deployment web --image=nginx --replicas=3      │   │
│   │  kubectl scale deployment web --replicas=5                     │   │
│   │  kubectl set image deployment/web nginx=nginx:1.19             │   │
│   │  kubectl rollout status deployment/web                         │   │
│   │  kubectl rollout undo deployment/web                           │   │
│   │  kubectl rollout history deployment/web                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   SERVICES:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl expose deployment web --port=80 --target-port=8080    │   │
│   │  kubectl expose pod nginx --port=80 --type=NodePort            │   │
│   │  kubectl get endpoints <service>                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   NODE OPERATIONS:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl drain <node> --ignore-daemonsets --force              │   │
│   │  kubectl cordon <node>                                         │   │
│   │  kubectl uncordon <node>                                       │   │
│   │  kubectl taint nodes <node> key=value:NoSchedule               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   TROUBLESHOOTING:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe pod <pod>         # Check Events             │   │
│   │  kubectl logs <pod> --previous      # Previous container       │   │
│   │  kubectl get events --sort-by='.lastTimestamp'                 │   │
│   │  kubectl top pods / nodes                                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 etcd Backup (MEMORIZE!)

```bash
# This command format appears frequently!
ETCDCTL_API=3 etcdctl snapshot save /path/to/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /path/to/backup.db
```

### 4.3 RBAC Quick Reference

```bash
# Create Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods

# Create RoleBinding
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=jane

# Create ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create ClusterRoleBinding
kubectl create clusterrolebinding read-nodes \
  --clusterrole=node-reader \
  --user=jane

# Test permissions
kubectl auth can-i get pods --as jane
kubectl auth can-i get pods --as jane -n default
```

---

## 5. Time Management

### 5.1 Question Strategy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TIME MANAGEMENT                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   2 hours / ~17 questions = ~7 minutes per question                     │
│                                                                         │
│   STRATEGY:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   First Pass (90 minutes):                                      │   │
│   │   • Do all EASY questions first                                 │   │
│   │   • Skip hard questions (flag them)                             │   │
│   │   • Don't spend more than 5 min on any question                │   │
│   │                                                                 │   │
│   │   Second Pass (30 minutes):                                     │   │
│   │   • Return to flagged questions                                 │   │
│   │   • Attempt partially if time short                            │   │
│   │   • Partial credit is possible!                                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   QUESTION WEIGHTS:                                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Each question has a weight (e.g., 4%, 7%, 13%)               │   │
│   │                                                                 │   │
│   │   • High-weight questions (10%+): Spend more time              │   │
│   │   • Low-weight questions (2-4%): Don't overthink               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Per-Question Checklist

```
1. ☐ Read the ENTIRE question carefully
2. ☐ Note the cluster context (switch if needed)
3. ☐ Note the namespace
4. ☐ Complete the task
5. ☐ VERIFY your work (kubectl get, describe)
6. ☐ Move to next question
```

---

## 6. Common Exam Tasks

### 6.1 Task Types

| Task Type | Example |
|-----------|---------|
| Create Pod | "Create a pod with specific image and labels" |
| Create Deployment | "Deploy nginx with 3 replicas" |
| Expose Service | "Expose deployment on NodePort 30080" |
| Troubleshoot | "Pod is not starting, fix it" |
| RBAC | "Create role with specific permissions" |
| Network Policy | "Allow traffic only from specific pods" |
| Storage | "Create PV and PVC, mount in pod" |
| Upgrade | "Upgrade cluster to version X" |
| etcd | "Backup etcd to specific location" |
| Node | "Drain node for maintenance" |
| Logs | "Get logs and save to file" |
| Scheduling | "Schedule pod on specific node" |

### 6.2 Sample Tasks

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SAMPLE EXAM TASKS                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Task 1: Create a Pod (4%)                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  "Create a pod named 'web' with image nginx:1.19 in namespace  │   │
│   │   'dev'. Add label app=web."                                    │   │
│   │                                                                 │   │
│   │  Solution:                                                      │   │
│   │  kubectl run web --image=nginx:1.19 -n dev -l app=web          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Task 2: Troubleshoot Pod (7%)                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  "Pod 'broken-pod' in namespace 'prod' is not running.         │   │
│   │   Identify and fix the issue."                                  │   │
│   │                                                                 │   │
│   │  Solution:                                                      │   │
│   │  kubectl describe pod broken-pod -n prod                       │   │
│   │  kubectl logs broken-pod -n prod --previous                    │   │
│   │  # Fix based on error (edit pod/deployment)                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Task 3: RBAC (7%)                                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  "Create a ServiceAccount 'deploy-sa'. Create a Role that      │   │
│   │   allows get,list,create deployments. Bind role to SA."        │   │
│   │                                                                 │   │
│   │  Solution:                                                      │   │
│   │  kubectl create sa deploy-sa                                   │   │
│   │  kubectl create role deploy-role --verb=get,list,create \      │   │
│   │    --resource=deployments                                      │   │
│   │  kubectl create rolebinding deploy-rb \                        │   │
│   │    --role=deploy-role --serviceaccount=default:deploy-sa       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Task 4: etcd Backup (7%)                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  "Backup etcd to /opt/backup/etcd-snapshot.db"                 │   │
│   │                                                                 │   │
│   │  Solution:                                                      │   │
│   │  ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd.db \     │   │
│   │    --endpoints=https://127.0.0.1:2379 \                        │   │
│   │    --cacert=/etc/kubernetes/pki/etcd/ca.crt \                  │   │
│   │    --cert=/etc/kubernetes/pki/etcd/server.crt \                │   │
│   │    --key=/etc/kubernetes/pki/etcd/server.key                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Task 5: NetworkPolicy (7%)                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  "Create NetworkPolicy 'allow-web' that allows ingress to      │   │
│   │   pods with label app=web only from pods with label role=api"  │   │
│   │                                                                 │   │
│   │  Solution: Create YAML with correct spec                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Documentation Navigation

### 7.1 Allowed Documentation

```
Only ONE browser tab with kubernetes.io is allowed:
• https://kubernetes.io/docs
• https://kubernetes.io/blog

Bookmark these pages:
• kubectl cheat sheet
• Pod spec reference  
• Deployment reference
• Service reference
• PV/PVC reference
• NetworkPolicy reference
```

### 7.2 Quick Doc Search

```bash
# In the exam, use the docs search bar effectively:
# Search for: "networkpolicy" → Copy YAML example
# Search for: "persistent volume" → Find PV/PVC examples
# Search for: "pod security context" → Find securityContext
```

---

## 8. Practice Resources

### 8.1 Recommended Practice

| Resource | Type | Notes |
|----------|------|-------|
| **Killer.sh** | Simulator | 2 free sessions with exam purchase |
| **KodeKloud** | Course + Labs | Great hands-on practice |
| **Kubernetes.io Tasks** | Docs | Official tutorials |
| **Play with Kubernetes** | Free Lab | Browser-based K8s |
| **Minikube/kind** | Local | Practice on your machine |

### 8.2 Practice Exercises

```bash
# Exercise 1: Create and scale
kubectl create deployment web --image=nginx --replicas=3
kubectl scale deployment web --replicas=5
kubectl expose deployment web --port=80 --type=NodePort

# Exercise 2: Troubleshoot
# Create a broken pod, then fix it
kubectl run broken --image=nginx:wrong-tag
kubectl describe pod broken
kubectl set image pod/broken broken=nginx:latest

# Exercise 3: RBAC
kubectl create sa mysa
kubectl create role myrole --verb=get,list --resource=pods
kubectl create rolebinding myrb --role=myrole --serviceaccount=default:mysa
kubectl auth can-i get pods --as system:serviceaccount:default:mysa

# Exercise 4: etcd backup (on control plane)
ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Exercise 5: Node maintenance
kubectl drain <node> --ignore-daemonsets
kubectl uncordon <node>
```

---

## 9. Day of Exam

### 9.1 Before the Exam

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRE-EXAM CHECKLIST                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Environment:                                                          │
│   ☐ Quiet room with door                                               │
│   ☐ Clear desk (nothing except laptop)                                 │
│   ☐ Government-issued ID ready                                         │
│   ☐ Stable internet connection                                         │
│   ☐ Webcam and microphone working                                      │
│   ☐ External monitors disconnected                                     │
│                                                                         │
│   Technical:                                                            │
│   ☐ Chrome or Chromium browser                                         │
│   ☐ PSI Secure Browser installed                                       │
│   ☐ System requirements verified                                       │
│   ☐ Tested webcam with proctoring software                            │
│                                                                         │
│   Mental:                                                               │
│   ☐ Good night's sleep                                                 │
│   ☐ Light meal before exam                                             │
│   ☐ Use bathroom before starting                                       │
│   ☐ Stay calm - you have 2 hours!                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 During the Exam

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DURING EXAM TIPS                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   First 5 Minutes:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  1. Verify aliases work: alias k=kubectl                        │   │
│   │  2. Set up vim: echo "set ts=2 sw=2 et" >> ~/.vimrc            │   │
│   │  3. Read first few questions to plan                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   For Each Question:                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  1. Switch to correct cluster context!                          │   │
│   │  2. Read question completely                                    │   │
│   │  3. Note namespace requirement                                  │   │
│   │  4. Complete task                                               │   │
│   │  5. Verify with kubectl get/describe                           │   │
│   │  6. Flag if unsure and move on                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Important:                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • ALWAYS switch context per question                          │   │
│   │  • Watch the timer but don't panic                             │   │
│   │  • Partial solutions get partial credit                        │   │
│   │  • You can flag and return to questions                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Common Mistakes to Avoid

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON MISTAKES                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ❌ WRONG CONTEXT                                                      │
│   • Forgetting to switch clusters between questions                    │
│   • Solution: ALWAYS check context first                               │
│                                                                         │
│   ❌ WRONG NAMESPACE                                                    │
│   • Creating resources in wrong namespace                              │
│   • Solution: Use -n <namespace> or set default                        │
│                                                                         │
│   ❌ NOT VERIFYING                                                      │
│   • Assuming command worked without checking                           │
│   • Solution: Always kubectl get/describe after                        │
│                                                                         │
│   ❌ SPENDING TOO LONG                                                  │
│   • Stuck on one question for 15+ minutes                             │
│   • Solution: Flag and move on after 5-7 minutes                       │
│                                                                         │
│   ❌ TYPOS                                                              │
│   • Wrong pod name, image name, namespace                             │
│   • Solution: Copy names from question, use tab completion             │
│                                                                         │
│   ❌ NOT READING FULLY                                                  │
│   • Missing requirements in long question text                         │
│   • Solution: Read entire question, note all requirements              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Final Checklist

### 11.1 Topics Mastery Checklist

```
Core Skills:
☐ Create pods, deployments, services imperatively
☐ Generate YAML with --dry-run=client -o yaml
☐ Edit running resources with kubectl edit
☐ Scale deployments
☐ Rolling updates and rollbacks

Troubleshooting:
☐ Debug pending/crashing pods
☐ Check logs (including --previous)
☐ Understand pod status meanings
☐ Fix node issues (kubelet)
☐ Service endpoint issues

Cluster Admin:
☐ etcd backup and restore
☐ Cluster upgrade process
☐ Node drain/cordon/uncordon
☐ RBAC (Role, RoleBinding, ClusterRole)
☐ Certificate expiration check

Networking:
☐ Service types (ClusterIP, NodePort, LoadBalancer)
☐ Ingress resources
☐ NetworkPolicy (ingress/egress rules)
☐ DNS debugging

Storage:
☐ PersistentVolume creation
☐ PersistentVolumeClaim
☐ StorageClass
☐ Mount volumes in pods

Scheduling:
☐ nodeSelector
☐ Node affinity
☐ Taints and tolerations
☐ Resource requests/limits
```

---

## 12. Quick Command Reference

```bash
# === PODS ===
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod> [--previous]
kubectl exec -it <pod> -- /bin/sh
kubectl delete pod <pod> --force --grace-period=0

# === DEPLOYMENTS ===
kubectl create deployment web --image=nginx --replicas=3
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.19
kubectl rollout status deployment/web
kubectl rollout undo deployment/web
kubectl rollout history deployment/web

# === SERVICES ===
kubectl expose deployment web --port=80 --type=NodePort
kubectl get svc
kubectl get endpoints <svc>

# === CONFIGMAPS & SECRETS ===
kubectl create configmap my-cm --from-literal=key=value
kubectl create secret generic my-secret --from-literal=pass=secret

# === RBAC ===
kubectl create sa <name>
kubectl create role <name> --verb=get,list --resource=pods
kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa>
kubectl auth can-i get pods --as <user>

# === NODES ===
kubectl drain <node> --ignore-daemonsets --force
kubectl cordon <node>
kubectl uncordon <node>
kubectl taint nodes <node> key=value:NoSchedule

# === ETCD ===
ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# === TROUBLESHOOTING ===
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods
kubectl top nodes
```

---

## 🎉 Congratulations!

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   ╔═══════════════════════════════════════════════════════════════╗     │
│   ║                                                               ║     │
│   ║        YOU'VE COMPLETED THE KUBERNETES LEARNING PATH!        ║     │
│   ║                                                               ║     │
│   ║   29 Comprehensive Chapters                                   ║     │
│   ║   1.5+ MB of Learning Material                               ║     │
│   ║   30,000+ Lines of Content                                   ║     │
│   ║   100+ ASCII Diagrams                                        ║     │
│   ║   200+ YAML Examples                                         ║     │
│   ║   500+ Commands                                              ║     │
│   ║                                                               ║     │
│   ║   You are now ready for the CKA Exam!                        ║     │
│   ║                                                               ║     │
│   ╚═══════════════════════════════════════════════════════════════╝     │
│                                                                         │
│   NEXT STEPS:                                                           │
│   1. Practice with Killer.sh (2 free sessions with exam)               │
│   2. Practice daily with local cluster (minikube/kind)                 │
│   3. Schedule your exam when ready                                     │
│   4. Pass and get certified! 🏆                                        │
│                                                                         │
│   GOOD LUCK ON YOUR CKA EXAM!                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**Chapter 29 Complete! 🎉**

**YOUR KUBERNETES LEARNING JOURNEY IS COMPLETE!**

You now have:
- Complete Docker knowledge (16 chapters)
- Complete Kubernetes knowledge (29 chapters)
- CKA exam preparation
- All the tools to pass the certification!

**GO ACE THAT CKA EXAM! 🏆**

