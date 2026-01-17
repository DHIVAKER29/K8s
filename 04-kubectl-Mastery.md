# ☸️ Chapter 04: kubectl Mastery

> Master the Kubernetes command-line tool - your primary interface to the cluster.

---

## 📚 Table of Contents

1. [What is kubectl?](#-what-is-kubectl)
2. [kubectl Syntax](#-kubectl-syntax)
3. [Essential Commands](#-essential-commands)
4. [Resource Management](#-resource-management)
5. [Output Formatting](#-output-formatting)
6. [Filtering and Selecting](#-filtering-and-selecting)
7. [Imperative vs Declarative](#-imperative-vs-declarative)
8. [Working with YAML](#-working-with-yaml)
9. [Debugging and Troubleshooting](#-debugging-and-troubleshooting)
10. [Advanced kubectl](#-advanced-kubectl)
11. [Productivity Tips](#-productivity-tips)
12. [CKA Exam Speed Tips](#-cka-exam-speed-tips)
13. [Command Reference](#-command-reference)
14. [Summary](#-summary)

---

## 📖 What is kubectl?

### Definition

> **kubectl** (pronounced "cube-control", "cube-c-t-l", or "cube-cuddle") is the command-line interface for running commands against Kubernetes clusters.

### How kubectl Works

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            HOW KUBECTL WORKS                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────┐      ┌──────────────────────┐      ┌───────────────────────────┐ │
│  │    User      │      │      kubectl         │      │      API Server           │ │
│  │              │      │                      │      │                           │ │
│  │  kubectl     │─────>│  1. Read kubeconfig  │─────>│  4. Authenticate          │ │
│  │  get pods    │      │  2. Build HTTP req   │      │  5. Authorize             │ │
│  │              │      │  3. Send to API      │<─────│  6. Return response       │ │
│  │              │<─────│  7. Format output    │      │                           │ │
│  └──────────────┘      └──────────────────────┘      └───────────────────────────┘ │
│                                                                                      │
│  kubeconfig (~/.kube/config) contains:                                              │
│  • Cluster endpoints (API server URL)                                               │
│  • Authentication credentials (certs, tokens)                                       │
│  • Context (which cluster + user + namespace)                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Verify Installation

```bash
# Check kubectl version
kubectl version

# Client and server version
kubectl version --short

# Only client version (no cluster needed)
kubectl version --client

# Check cluster connectivity
kubectl cluster-info
```

---

## 📝 kubectl Syntax

### Basic Syntax

```
kubectl [command] [TYPE] [NAME] [flags]

Where:
• command   = What to do (get, create, delete, apply, describe, etc.)
• TYPE      = Resource type (pod, deployment, service, etc.)
• NAME      = Resource name (optional, omit for all resources)
• flags     = Optional flags (-n namespace, -o output, etc.)
```

### Examples

```bash
# Get all pods
kubectl get pods

# Get specific pod
kubectl get pod nginx-pod

# Get pods in namespace
kubectl get pods -n kube-system

# Get pods with specific output
kubectl get pods -o wide

# Multiple resources
kubectl get pods,services

# All resources
kubectl get all
```

### Resource Type Shortcuts

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         RESOURCE TYPE SHORTCUTS                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Full Name              │ Short   │ API Group                                       │
│  ───────────────────────┼─────────┼─────────────────────────────────────────────────│
│  pods                   │ po      │ core (v1)                                       │
│  services               │ svc     │ core (v1)                                       │
│  deployments            │ deploy  │ apps/v1                                         │
│  replicasets            │ rs      │ apps/v1                                         │
│  statefulsets           │ sts     │ apps/v1                                         │
│  daemonsets             │ ds      │ apps/v1                                         │
│  configmaps             │ cm      │ core (v1)                                       │
│  secrets                │ -       │ core (v1)                                       │
│  persistentvolumes      │ pv      │ core (v1)                                       │
│  persistentvolumeclaims │ pvc     │ core (v1)                                       │
│  namespaces             │ ns      │ core (v1)                                       │
│  nodes                  │ no      │ core (v1)                                       │
│  endpoints              │ ep      │ core (v1)                                       │
│  events                 │ ev      │ core (v1)                                       │
│  ingresses              │ ing     │ networking.k8s.io/v1                           │
│  networkpolicies        │ netpol  │ networking.k8s.io/v1                           │
│  serviceaccounts        │ sa      │ core (v1)                                       │
│  horizontalpodautoscalers│ hpa    │ autoscaling/v2                                  │
│  cronjobs               │ cj      │ batch/v1                                        │
│  jobs                   │ -       │ batch/v1                                        │
│                                                                                      │
│  Get all resource types:                                                            │
│  kubectl api-resources                                                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚡ Essential Commands

### CRUD Operations

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          CRUD OPERATIONS                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  CREATE                                                                             │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl create -f manifest.yaml          # Create from file                        │
│  kubectl apply -f manifest.yaml           # Create or update from file              │
│  kubectl create deployment nginx --image=nginx  # Create imperatively              │
│  kubectl run nginx --image=nginx          # Create pod                              │
│                                                                                      │
│  READ                                                                               │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl get pods                         # List pods                               │
│  kubectl get pods -o wide                 # More details                            │
│  kubectl describe pod nginx               # Full details                            │
│  kubectl get pod nginx -o yaml            # YAML format                             │
│                                                                                      │
│  UPDATE                                                                             │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl apply -f manifest.yaml           # Apply changes                           │
│  kubectl edit deployment nginx            # Edit in editor                          │
│  kubectl set image deploy/nginx nginx=nginx:1.19  # Update image                   │
│  kubectl scale deploy nginx --replicas=5  # Scale                                   │
│  kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'                       │
│                                                                                      │
│  DELETE                                                                             │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl delete pod nginx                 # Delete specific                         │
│  kubectl delete -f manifest.yaml          # Delete from file                        │
│  kubectl delete pods --all                # Delete all pods                         │
│  kubectl delete pods -l app=nginx         # Delete by label                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Get Command Variations

```bash
# Basic get
kubectl get pods                    # List all pods in current namespace
kubectl get pods -n kube-system     # Specific namespace
kubectl get pods --all-namespaces   # All namespaces (or -A)
kubectl get pods -A                 # Short form

# Get specific resource
kubectl get pod nginx               # Single pod
kubectl get pods nginx1 nginx2      # Multiple pods

# Get multiple resource types
kubectl get pods,services           # Pods and services
kubectl get all                     # Common resources (not really all!)

# Output formats
kubectl get pods -o wide            # Extra columns (IP, Node)
kubectl get pods -o yaml            # YAML format
kubectl get pods -o json            # JSON format
kubectl get pods -o name            # Just names

# Watch for changes
kubectl get pods -w                 # Watch mode (live updates)
kubectl get pods --watch            # Same as -w

# Sort and filter
kubectl get pods --sort-by=.metadata.name
kubectl get pods --sort-by=.status.startTime
kubectl get pods --field-selector=status.phase=Running
```

### Describe Command

```bash
# Describe shows detailed information including events
kubectl describe pod nginx
kubectl describe node worker-1
kubectl describe deployment nginx
kubectl describe service my-service

# Describe is essential for troubleshooting!
# Look for:
# - Status and conditions
# - Events (at the bottom)
# - Resource requests/limits
# - Volume mounts
# - Container statuses
```

### Logs Command

```bash
# View container logs
kubectl logs nginx                  # Pod logs
kubectl logs nginx -c container1    # Specific container
kubectl logs nginx --all-containers # All containers in pod

# Follow logs (live)
kubectl logs -f nginx               # Follow/stream
kubectl logs nginx --follow         # Same as -f

# Historical logs
kubectl logs nginx --tail=100       # Last 100 lines
kubectl logs nginx --since=1h       # Last hour
kubectl logs nginx --since=30m      # Last 30 minutes
kubectl logs nginx --timestamps     # With timestamps

# Previous container (after restart)
kubectl logs nginx --previous       # Previous instance
kubectl logs nginx -p               # Short form

# Logs from multiple pods
kubectl logs -l app=nginx           # By label selector
kubectl logs -l app=nginx --all-containers
```

### Exec Command

```bash
# Execute command in container
kubectl exec nginx -- ls /           # Run command
kubectl exec nginx -- cat /etc/hosts

# Interactive shell
kubectl exec -it nginx -- /bin/bash  # Bash shell
kubectl exec -it nginx -- /bin/sh    # Shell (for alpine)
kubectl exec -it nginx -- sh         # Short

# Specific container (multi-container pod)
kubectl exec -it nginx -c sidecar -- /bin/sh

# Run command and exit
kubectl exec nginx -- env            # View environment
kubectl exec nginx -- whoami         # Current user
kubectl exec nginx -- ps aux         # Processes
```

### Port Forward

```bash
# Forward local port to pod port
kubectl port-forward pod/nginx 8080:80

# Forward to service
kubectl port-forward svc/nginx 8080:80

# Forward to deployment
kubectl port-forward deployment/nginx 8080:80

# Listen on all interfaces
kubectl port-forward --address 0.0.0.0 pod/nginx 8080:80

# Background (in shell)
kubectl port-forward pod/nginx 8080:80 &
```

### Copy Files

```bash
# Copy from pod to local
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp default/nginx:/app/data ./data

# Copy from local to pod
kubectl cp ./config.yaml nginx:/app/config.yaml

# With specific container
kubectl cp nginx:/logs ./logs -c sidecar

# Note: Requires tar in container
```

---

## 📋 Resource Management

### Namespace Operations

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create namespace
kubectl create namespace dev
kubectl create ns staging

# Set default namespace
kubectl config set-context --current --namespace=dev

# Delete namespace (and all resources in it!)
kubectl delete namespace dev

# Run command in specific namespace
kubectl get pods -n kube-system
kubectl get all -n production
```

### Labels and Annotations

```bash
# Add label
kubectl label pod nginx environment=production
kubectl label pods --all tier=frontend

# Update label (overwrite)
kubectl label pod nginx environment=staging --overwrite

# Remove label
kubectl label pod nginx environment-

# Add annotation
kubectl annotate pod nginx description="Web server"

# Select by label
kubectl get pods -l app=nginx
kubectl get pods -l 'app=nginx,tier=frontend'
kubectl get pods -l 'app in (nginx,apache)'
kubectl get pods -l 'app notin (test)'
kubectl get pods -l 'environment!=production'

# Show labels
kubectl get pods --show-labels
kubectl get pods -L app,tier    # Specific labels as columns
```

### Resource Quotas and Limits

```bash
# View resource usage
kubectl top pods                    # Pod CPU/memory (requires metrics-server)
kubectl top nodes                   # Node CPU/memory
kubectl top pod nginx --containers  # Container level

# View resource quotas
kubectl get resourcequotas
kubectl describe resourcequota my-quota

# View limit ranges
kubectl get limitranges
kubectl describe limitrange my-limits
```

---

## 📊 Output Formatting

### Output Formats (-o flag)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          OUTPUT FORMATS                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Format              │ Description                    │ Example                     │
│  ────────────────────┼────────────────────────────────┼─────────────────────────────│
│  -o wide             │ Additional columns             │ kubectl get pods -o wide    │
│  -o yaml             │ YAML format                    │ kubectl get pod nginx -o yaml│
│  -o json             │ JSON format                    │ kubectl get pod nginx -o json│
│  -o name             │ Just resource names            │ kubectl get pods -o name    │
│  -o jsonpath='{}'    │ JSONPath expression            │ See below                   │
│  -o custom-columns=  │ Custom column output           │ See below                   │
│  -o go-template=     │ Go template                    │ Advanced                    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### JSONPath (Essential for CKA!)

```bash
# Basic JSONPath
kubectl get pod nginx -o jsonpath='{.metadata.name}'
# Output: nginx

# Get multiple fields
kubectl get pod nginx -o jsonpath='{.metadata.name} {.status.phase}'
# Output: nginx Running

# Get from list
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
# Output: nginx1 nginx2 nginx3

# With newlines
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
# Output:
# nginx1
# nginx2
# nginx3

# Get container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Get specific container image
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'

# Get node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Get pod IPs
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

### JSONPath Syntax Reference

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          JSONPATH SYNTAX                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Syntax              │ Meaning                                                      │
│  ────────────────────┼──────────────────────────────────────────────────────────────│
│  .field              │ Access field                                                 │
│  ['field']           │ Access field (alternative)                                   │
│  [0]                 │ Array index                                                  │
│  [*]                 │ All elements                                                 │
│  [start:end]         │ Array slice                                                  │
│  [?(@.key==value)]   │ Filter                                                       │
│  ..                  │ Recursive descent                                            │
│  {"\n"}              │ Newline                                                      │
│  {"\t"}              │ Tab                                                          │
│  {range ...}{end}    │ Iterate                                                      │
│                                                                                      │
│  Examples:                                                                          │
│  ────────────────────────────────────────────────────────────────────────────────────│
│  .metadata.name                           # Pod name                                │
│  .spec.containers[0].image                # First container image                  │
│  .spec.containers[*].name                 # All container names                    │
│  .status.conditions[?(@.type=="Ready")]   # Ready condition                        │
│  .items[*].metadata.name                  # All pod names from list                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Custom Columns

```bash
# Basic custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# With header
kubectl get pods -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName'

# From file
cat columns.txt
NAME:.metadata.name
STATUS:.status.phase
IP:.status.podIP

kubectl get pods -o custom-columns-file=columns.txt

# Useful examples
kubectl get pods -o custom-columns=\
'POD:.metadata.name,IMAGE:.spec.containers[0].image,NODE:.spec.nodeName'

kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.conditions[?(@.type=="Ready")].status,VERSION:.status.nodeInfo.kubeletVersion'
```

---

## 🔍 Filtering and Selecting

### Field Selectors

```bash
# Filter by field values
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=status.phase!=Running
kubectl get pods --field-selector=metadata.name=nginx
kubectl get pods --field-selector=metadata.namespace=default

# Multiple conditions
kubectl get pods --field-selector=status.phase=Running,metadata.namespace=default

# Common field selectors
kubectl get pods --field-selector=spec.nodeName=worker-1
kubectl get events --field-selector=type=Warning
kubectl get events --field-selector=involvedObject.name=nginx
```

### Label Selectors

```bash
# Equality-based
kubectl get pods -l app=nginx
kubectl get pods -l app!=nginx
kubectl get pods -l 'app=nginx,tier=frontend'

# Set-based
kubectl get pods -l 'app in (nginx,apache)'
kubectl get pods -l 'app notin (nginx)'
kubectl get pods -l 'environment'          # Has label
kubectl get pods -l '!environment'         # Doesn't have label

# Selector with output
kubectl get pods -l app=nginx -o wide
kubectl get pods -l app=nginx --show-labels
```

### Sorting

```bash
# Sort by field
kubectl get pods --sort-by=.metadata.name
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime

# Sort events by time
kubectl get events --sort-by=.metadata.creationTimestamp

# Sort nodes by capacity
kubectl get nodes --sort-by=.status.capacity.memory
```

---

## 🔄 Imperative vs Declarative

### Imperative Commands (Quick Actions)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          IMPERATIVE COMMANDS                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  "Do this action right now"                                                         │
│                                                                                      │
│  Create Resources:                                                                  │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl run nginx --image=nginx                    # Create pod                    │
│  kubectl create deployment nginx --image=nginx     # Create deployment             │
│  kubectl create service clusterip nginx --tcp=80   # Create service               │
│  kubectl create configmap my-config --from-literal=key=value                      │
│  kubectl create secret generic my-secret --from-literal=password=pass             │
│  kubectl create namespace dev                       # Create namespace             │
│                                                                                      │
│  Modify Resources:                                                                  │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl scale deployment nginx --replicas=5        # Scale                        │
│  kubectl set image deploy/nginx nginx=nginx:1.19    # Update image                │
│  kubectl edit deployment nginx                      # Edit in editor              │
│  kubectl label pod nginx app=web                    # Add label                   │
│  kubectl annotate pod nginx desc="my pod"           # Add annotation              │
│                                                                                      │
│  Expose Resources:                                                                  │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl expose pod nginx --port=80                 # Expose pod                   │
│  kubectl expose deployment nginx --port=80 --type=NodePort                        │
│                                                                                      │
│  ✅ Good for: Quick testing, CKA exam, one-off tasks                               │
│  ❌ Not for: Production, version control, repeatability                            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Declarative Commands (YAML Files)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          DECLARATIVE APPROACH                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  "This is the state I want"                                                         │
│                                                                                      │
│  Apply manifests:                                                                   │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl apply -f pod.yaml                     # Apply single file                  │
│  kubectl apply -f deployment.yaml              # Apply deployment                   │
│  kubectl apply -f directory/                   # Apply all in directory             │
│  kubectl apply -f https://url/manifest.yaml   # Apply from URL                     │
│  kubectl apply -f file1.yaml -f file2.yaml    # Multiple files                     │
│  kubectl apply -k ./kustomize/                 # Kustomize                          │
│                                                                                      │
│  Delete from manifests:                                                             │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl delete -f pod.yaml                                                         │
│  kubectl delete -f directory/                                                       │
│                                                                                      │
│  Diff before apply:                                                                 │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  kubectl diff -f deployment.yaml               # Show what would change            │
│                                                                                      │
│  ✅ Good for: Production, GitOps, version control, repeatability                   │
│  ❌ Not for: Quick one-off testing                                                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Generate YAML from Imperative Commands (CKA Gold!)

```bash
# --dry-run=client -o yaml generates YAML without creating the resource
# This is ESSENTIAL for CKA exam!

# Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml

# Service (ClusterIP)
kubectl create service clusterip nginx --tcp=80:80 --dry-run=client -o yaml > svc.yaml

# Service (from deployment)
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml > svc.yaml

# Job
kubectl create job my-job --image=busybox --dry-run=client -o yaml > job.yaml

# CronJob
kubectl create cronjob my-cron --image=busybox --schedule="*/1 * * * *" --dry-run=client -o yaml > cronjob.yaml

# ConfigMap
kubectl create configmap my-config --from-literal=key=value --dry-run=client -o yaml > cm.yaml

# Secret
kubectl create secret generic my-secret --from-literal=pass=secret --dry-run=client -o yaml > secret.yaml

# ServiceAccount
kubectl create serviceaccount my-sa --dry-run=client -o yaml > sa.yaml

# Namespace
kubectl create namespace dev --dry-run=client -o yaml > ns.yaml
```

---

## 📄 Working with YAML

### YAML Structure

```yaml
# Every Kubernetes YAML has these 4 top-level fields
apiVersion: v1              # API version (v1, apps/v1, networking.k8s.io/v1)
kind: Pod                   # Resource type
metadata:                   # Resource metadata
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:                       # Resource specification (varies by kind)
  containers:
  - name: nginx
    image: nginx
```

### Get Resource YAML

```bash
# Get existing resource as YAML
kubectl get pod nginx -o yaml

# Get and save to file
kubectl get pod nginx -o yaml > pod.yaml

# Get without cluster-specific fields (cleaner)
kubectl get pod nginx -o yaml | kubectl neat   # Requires kubectl-neat plugin

# Or manually remove:
# - status
# - metadata.uid
# - metadata.resourceVersion
# - metadata.creationTimestamp
# - metadata.managedFields
```

### Edit Resources

```bash
# Edit in default editor
kubectl edit pod nginx
kubectl edit deployment nginx

# Set editor
export KUBE_EDITOR=vim
export KUBE_EDITOR="code --wait"  # VS Code

# Edit and apply pattern
kubectl get deployment nginx -o yaml > nginx.yaml
vim nginx.yaml
kubectl apply -f nginx.yaml
```

### Replace vs Apply vs Create

```bash
# create: Creates new resource (fails if exists)
kubectl create -f pod.yaml

# apply: Creates or updates (idempotent, recommended)
kubectl apply -f pod.yaml

# replace: Replaces entire resource (fails if doesn't exist)
kubectl replace -f pod.yaml

# replace --force: Delete and recreate
kubectl replace --force -f pod.yaml
```

---

## 🔧 Debugging and Troubleshooting

### Debugging Pods

```bash
# Check pod status
kubectl get pod nginx
kubectl get pod nginx -o wide

# Detailed information
kubectl describe pod nginx

# Check events (usually at bottom of describe)
kubectl get events --field-selector=involvedObject.name=nginx

# Check logs
kubectl logs nginx
kubectl logs nginx --previous      # Previous container
kubectl logs nginx -c container1   # Specific container

# Execute into pod
kubectl exec -it nginx -- /bin/sh

# Check resource usage
kubectl top pod nginx
```

### Common Pod Issues

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          COMMON POD ISSUES                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Status: Pending                                                                     │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • No nodes available (check: kubectl get nodes)                                    │
│  • Insufficient resources (check: kubectl describe pod)                             │
│  • Taints not tolerated (check: kubectl describe node)                             │
│  • Volume not bound (check: kubectl get pvc)                                        │
│                                                                                      │
│  Status: ImagePullBackOff / ErrImagePull                                            │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • Wrong image name                                                                 │
│  • Image doesn't exist                                                              │
│  • Private registry without credentials                                             │
│  • Network issues                                                                    │
│                                                                                      │
│  Status: CrashLoopBackOff                                                            │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • Application crashing (check: kubectl logs --previous)                            │
│  • Missing config/secrets                                                           │
│  • Wrong command/args                                                               │
│  • Health check failing                                                              │
│                                                                                      │
│  Status: Running but not working                                                     │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • Check logs: kubectl logs <pod>                                                   │
│  • Check connectivity: kubectl exec <pod> -- curl localhost                        │
│  • Check service: kubectl get endpoints                                             │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Debugging Commands

```bash
# Run debug pod
kubectl run debug --image=busybox:1.28 --rm -it -- /bin/sh

# Debug with network tools
kubectl run debug --image=nicolaka/netshoot --rm -it -- /bin/bash

# Check DNS
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes

# Check connectivity to service
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- http://my-service

# Debug node
kubectl debug node/worker-1 -it --image=ubuntu
```

---

## 🚀 Advanced kubectl

### Rollouts (Deployment Management)

```bash
# Check rollout status
kubectl rollout status deployment nginx

# View rollout history
kubectl rollout history deployment nginx
kubectl rollout history deployment nginx --revision=2

# Undo rollout
kubectl rollout undo deployment nginx
kubectl rollout undo deployment nginx --to-revision=1

# Pause/resume rollout
kubectl rollout pause deployment nginx
kubectl rollout resume deployment nginx

# Restart deployment (recreate all pods)
kubectl rollout restart deployment nginx
```

### Autoscaling

```bash
# Create HPA
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# View HPA
kubectl get hpa
kubectl describe hpa nginx

# Delete HPA
kubectl delete hpa nginx
```

### Cluster Management

```bash
# View cluster info
kubectl cluster-info
kubectl cluster-info dump    # Full dump (very verbose)

# View API resources
kubectl api-resources        # All resources
kubectl api-resources --namespaced=true   # Namespaced only
kubectl api-resources --namespaced=false  # Cluster-scoped

# View API versions
kubectl api-versions

# Check permissions
kubectl auth can-i create pods
kubectl auth can-i delete deployments
kubectl auth can-i '*' '*'                # All permissions
kubectl auth can-i get pods --as=user1    # As another user
```

### Patch Resources

```bash
# JSON patch
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'

# Strategic merge patch
kubectl patch deployment nginx --type=strategic -p '
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
'

# JSON merge patch
kubectl patch deployment nginx --type=merge -p '{"spec":{"replicas":3}}'
```

---

## ⚡ Productivity Tips

### Shell Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc

# Basic shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kgpa='kubectl get pods -A'

# Describe
alias kdp='kubectl describe pod'
alias kds='kubectl describe service'
alias kdd='kubectl describe deployment'

# Logs
alias kl='kubectl logs'
alias klf='kubectl logs -f'

# Apply/Delete
alias ka='kubectl apply -f'
alias kdel='kubectl delete -f'

# Context/Namespace
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'

# Quick run
alias krun='kubectl run --rm -it debug --image=busybox:1.28 -- /bin/sh'

# Dry run
alias kdry='--dry-run=client -o yaml'
```

### Bash/Zsh Completion

```bash
# Bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Zsh
source <(kubectl completion zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc

# With alias
complete -o default -F __start_kubectl k   # bash
compdef k=kubectl                           # zsh
```

### kubectl Plugins (krew)

```bash
# Install krew (plugin manager)
# See: https://krew.sigs.k8s.io/docs/user-guide/setup/install/

# Useful plugins
kubectl krew install ctx       # Context switcher
kubectl krew install ns        # Namespace switcher
kubectl krew install neat      # Clean YAML output
kubectl krew install tree      # Show resource hierarchy
kubectl krew install images    # Show images in use
kubectl krew install access-matrix  # RBAC access matrix

# Use plugins
kubectl ctx           # Switch context interactively
kubectl ns            # Switch namespace interactively
```

---

## 🎓 CKA Exam Speed Tips

### Essential Speed Commands

```bash
# 1. Generate YAML quickly (MEMORIZE THIS!)
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml

# 2. Quick pod for testing
kubectl run test --image=busybox:1.28 --rm -it -- /bin/sh

# 3. Quick service creation
kubectl expose pod nginx --port=80
kubectl expose deployment nginx --port=80 --type=NodePort

# 4. Set context/namespace (do this first!)
kubectl config set-context --current --namespace=my-namespace

# 5. Quick scale
kubectl scale deployment nginx --replicas=5

# 6. Quick edit
kubectl edit deployment nginx

# 7. Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# 8. Quick logs
kubectl logs nginx --tail=50

# 9. Quick describe
kubectl describe pod nginx | less

# 10. Quick delete
kubectl delete pod nginx --force --grace-period=0
```

### CKA Exam Setup

```bash
# At start of exam, set up aliases:
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kga='kubectl get all'

# Enable completion
source <(kubectl completion bash)
complete -F __start_kubectl k

# Set default editor
export KUBE_EDITOR=vim

# Vim settings for YAML
echo 'set tabstop=2
set shiftwidth=2
set expandtab
set autoindent' >> ~/.vimrc
```

### Common CKA Patterns

```bash
# Pattern 1: Create pod with specific requirements
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
vim pod.yaml  # Add resources, volumes, etc.
kubectl apply -f pod.yaml

# Pattern 2: Expose and access
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx  # Get port

# Pattern 3: Debug failing pod
kubectl describe pod failing-pod
kubectl logs failing-pod --previous
kubectl get events | grep failing-pod

# Pattern 4: Update deployment image
kubectl set image deployment/nginx nginx=nginx:1.19

# Pattern 5: Rollback
kubectl rollout undo deployment/nginx

# Pattern 6: Create secret and use
kubectl create secret generic my-secret --from-literal=password=pass
# Then mount in pod spec

# Pattern 7: Create configmap and use
kubectl create configmap my-config --from-literal=key=value
# Then mount in pod spec
```

---

## 📚 Command Reference

### Complete Command List

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          KUBECTL COMMANDS REFERENCE                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Basic Commands:                                                                    │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  create        Create a resource                                                    │
│  get           Display resources                                                    │
│  run           Run a pod                                                           │
│  expose        Expose a resource as a service                                      │
│  delete        Delete resources                                                    │
│                                                                                      │
│  Deploy Commands:                                                                   │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  rollout       Manage rollouts                                                     │
│  scale         Scale a resource                                                    │
│  autoscale     Auto-scale a resource                                              │
│                                                                                      │
│  Cluster Management:                                                                │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  certificate   Modify certificate resources                                        │
│  cluster-info  Display cluster info                                               │
│  top           Display resource usage                                              │
│  cordon        Mark node as unschedulable                                          │
│  uncordon      Mark node as schedulable                                            │
│  drain         Drain node for maintenance                                          │
│  taint         Update taints on nodes                                              │
│                                                                                      │
│  Troubleshooting:                                                                   │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  describe      Show details of a resource                                          │
│  logs          Print container logs                                                │
│  exec          Execute command in container                                        │
│  port-forward  Forward local port to pod                                           │
│  cp            Copy files to/from containers                                       │
│  attach        Attach to running container                                         │
│  debug         Create debug container                                              │
│                                                                                      │
│  Advanced:                                                                          │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  apply         Apply configuration                                                 │
│  patch         Patch a resource                                                    │
│  replace       Replace a resource                                                  │
│  edit          Edit a resource                                                     │
│  label         Update labels                                                       │
│  annotate      Update annotations                                                  │
│                                                                                      │
│  Configuration:                                                                     │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  config        Modify kubeconfig                                                   │
│  set           Set specific features                                               │
│  wait          Wait for condition                                                  │
│                                                                                      │
│  Other:                                                                             │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  api-resources List API resources                                                  │
│  api-versions  List API versions                                                   │
│  auth          Inspect authorization                                               │
│  diff          Diff live vs applied                                               │
│  explain       Documentation for resources                                         │
│  version       Print version                                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### kubectl explain (Built-in Documentation)

```bash
# Explain resource fields
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources

# Recursive (show all fields)
kubectl explain pod --recursive

# Specific API version
kubectl explain pod --api-version=v1

# Very useful for CKA!
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.volumes
kubectl explain pod.spec.affinity
```

---

## ✅ Summary

### Key Commands to Master

| Category | Commands |
|----------|----------|
| **Get/List** | `get`, `-o wide`, `-o yaml`, `-o jsonpath` |
| **Details** | `describe`, `explain` |
| **Create** | `run`, `create`, `apply` |
| **Modify** | `edit`, `patch`, `set`, `scale` |
| **Delete** | `delete`, `delete -f` |
| **Debug** | `logs`, `exec`, `port-forward`, `cp` |
| **Rollouts** | `rollout status/history/undo` |
| **Config** | `config use-context`, `config set-context` |

### CKA Must-Know

```bash
# Generate YAML
kubectl <command> --dry-run=client -o yaml

# JSONPath for specific data
kubectl get <resource> -o jsonpath='{...}'

# Quick namespace switch
kubectl config set-context --current --namespace=<ns>

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Force delete
kubectl delete pod <name> --force --grace-period=0
```

---

## 🔜 What's Next

In **Chapter 05: Kubernetes Objects & YAML**, we'll cover:

- Understanding Kubernetes API objects
- YAML manifest structure in depth
- Common fields and their meanings
- Labels, selectors, and annotations
- Multi-resource manifests
- Best practices for YAML files

---

*kubectl is your primary tool - practice until these commands are muscle memory!*

