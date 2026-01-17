# Chapter 26: Logging & Monitoring

## Introduction

"What's happening in my application?" is a question you'll ask constantly. **Logging** captures events and errors. **Monitoring** tracks metrics and health. Together, they provide **observability** into your Kubernetes workloads.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
│   │     LOGS        │  │    METRICS      │  │    TRACES       │        │
│   │                 │  │                 │  │                 │        │
│   │  "What          │  │  "How is it     │  │  "Where did     │        │
│   │   happened?"    │  │   performing?"  │  │   time go?"     │        │
│   │                 │  │                 │  │                 │        │
│   │  Events         │  │  Numbers        │  │  Request flow   │        │
│   │  Errors         │  │  CPU/Memory     │  │  Latency        │        │
│   │  Debug info     │  │  Request rate   │  │  Dependencies   │        │
│   │                 │  │                 │  │                 │        │
│   │  kubectl logs   │  │  kubectl top    │  │  Jaeger/Zipkin  │        │
│   │  Fluentd/Loki   │  │  Prometheus     │  │  (Advanced)     │        │
│   │                 │  │                 │  │                 │        │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘        │
│          │                    │                    │                    │
│          └────────────────────┴────────────────────┘                    │
│                              │                                          │
│                    ┌─────────────────┐                                  │
│                    │  OBSERVABILITY  │                                  │
│                    │                 │                                  │
│                    │  Understanding  │                                  │
│                    │  system state   │                                  │
│                    │  from outputs   │                                  │
│                    │                 │                                  │
│                    └─────────────────┘                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Kubernetes Logging Architecture

### 1.1 How Container Logging Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  CONTAINER LOGGING FLOW                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                        POD                                      │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                    CONTAINER                            │   │   │
│   │   │                                                         │   │   │
│   │   │   Application                                           │   │   │
│   │   │      │                                                  │   │   │
│   │   │      ├──▶ stdout ──┐                                   │   │   │
│   │   │      │             │                                    │   │   │
│   │   │      └──▶ stderr ──┤                                   │   │   │
│   │   │                    │                                    │   │   │
│   │   └────────────────────│────────────────────────────────────┘   │   │
│   │                        │                                        │   │
│   └────────────────────────│────────────────────────────────────────┘   │
│                            │                                            │
│                            ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                  Container Runtime                              │   │
│   │                                                                 │   │
│   │   Captures stdout/stderr and writes to:                        │   │
│   │   /var/log/containers/<pod>_<ns>_<container>-<id>.log          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                            │                                            │
│                            ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                   Node Filesystem                               │   │
│   │                                                                 │   │
│   │   /var/log/containers/  ← Container logs                       │   │
│   │   /var/log/pods/        ← Symlinks                             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Log Locations

| Location | Contents |
|----------|----------|
| `/var/log/containers/` | Container logs (actual files) |
| `/var/log/pods/` | Symlinks to container logs |
| `/var/log/kubelet.log` | Kubelet logs |
| `/var/log/kube-proxy.log` | Kube-proxy logs |

### 1.3 Log Rotation

Kubernetes uses the container runtime's log rotation:

```yaml
# Kubelet configuration for log rotation
containerLogMaxSize: "10Mi"      # Max size per log file
containerLogMaxFiles: 5          # Number of log files to keep
```

---

## 2. kubectl logs

### 2.1 Basic Usage

```bash
# Get logs from a pod
kubectl logs <pod-name>

# Get logs from a specific container
kubectl logs <pod-name> -c <container-name>

# Get logs from previous container instance
kubectl logs <pod-name> --previous

# Follow logs in real-time
kubectl logs <pod-name> -f

# Get last N lines
kubectl logs <pod-name> --tail=100

# Get logs from last hour
kubectl logs <pod-name> --since=1h

# Get logs since specific time
kubectl logs <pod-name> --since-time="2024-01-15T10:00:00Z"
```

### 2.2 Multi-Container Pods

```bash
# List containers in pod
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'

# Get logs from specific container
kubectl logs <pod-name> -c <container-name>

# Get logs from all containers
kubectl logs <pod-name> --all-containers=true
```

### 2.3 Logs from Multiple Pods

```bash
# Logs from all pods with label
kubectl logs -l app=nginx

# Logs from all pods in deployment
kubectl logs deployment/web-app

# Logs from all pods with prefix
kubectl logs -l app=nginx --all-containers --prefix

# Follow logs from multiple pods
kubectl logs -l app=nginx -f
```

### 2.4 Output Options

```bash
# Add timestamps
kubectl logs <pod-name> --timestamps

# Limit bytes of output
kubectl logs <pod-name> --limit-bytes=10000

# Combine options
kubectl logs <pod-name> -f --tail=50 --timestamps
```

---

## 3. Cluster Component Logs

### 3.1 Control Plane Logs

```bash
# For kubeadm clusters (pods in kube-system)
kubectl logs -n kube-system <component-pod>

# API Server
kubectl logs -n kube-system kube-apiserver-<node>

# Controller Manager
kubectl logs -n kube-system kube-controller-manager-<node>

# Scheduler
kubectl logs -n kube-system kube-scheduler-<node>

# etcd
kubectl logs -n kube-system etcd-<node>
```

### 3.2 Node Component Logs

```bash
# Kubelet (systemd service, not pod)
journalctl -u kubelet

# Kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Container runtime (systemd)
journalctl -u containerd
journalctl -u docker
```

### 3.3 Log Verbosity

```bash
# API server verbosity levels
--v=0    # Minimal
--v=1    # Default
--v=2    # Useful info
--v=3    # Extended info
--v=4    # Debug level
--v=6    # API request/response
--v=8    # API request/response bodies
```

---

## 4. Kubernetes Events

### 4.1 What are Events?

Events are Kubernetes objects that record what happened:

```bash
# Get events in namespace
kubectl get events

# Get events sorted by time
kubectl get events --sort-by='.lastTimestamp'

# Get events for specific resource
kubectl get events --field-selector involvedObject.name=my-pod

# Watch events
kubectl get events -w
```

### 4.2 Event Structure

```
LAST SEEN   TYPE      REASON    OBJECT        MESSAGE
2m          Normal    Scheduled pod/web-app   Successfully assigned...
2m          Normal    Pulling   pod/web-app   Pulling image "nginx"
1m          Normal    Pulled    pod/web-app   Successfully pulled image
1m          Normal    Created   pod/web-app   Created container nginx
1m          Normal    Started   pod/web-app   Started container nginx
```

### 4.3 Event Types

| Type | Description |
|------|-------------|
| `Normal` | Routine operations |
| `Warning` | Something unusual |

### 4.4 Common Events

| Event | Meaning |
|-------|---------|
| `Scheduled` | Pod assigned to node |
| `Pulling` | Pulling image |
| `Pulled` | Image pulled successfully |
| `Created` | Container created |
| `Started` | Container started |
| `Killing` | Container being killed |
| `BackOff` | Container restarting |
| `FailedScheduling` | Can't schedule pod |
| `FailedMount` | Volume mount failed |
| `Unhealthy` | Probe failed |

---

## 5. Metrics Server

### 5.1 What is Metrics Server?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    METRICS SERVER                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Metrics Server collects resource metrics from kubelets                │
│                                                                         │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│   │   Node 1    │     │   Node 2    │     │   Node 3    │              │
│   │   kubelet   │     │   kubelet   │     │   kubelet   │              │
│   │     │       │     │     │       │     │     │       │              │
│   │   cAdvisor  │     │   cAdvisor  │     │   cAdvisor  │              │
│   │     │       │     │     │       │     │     │       │              │
│   └─────│───────┘     └─────│───────┘     └─────│───────┘              │
│         │                   │                   │                       │
│         └───────────────────┼───────────────────┘                       │
│                             │                                           │
│                             ▼                                           │
│                    ┌────────────────┐                                   │
│                    │ Metrics Server │                                   │
│                    │                │                                   │
│                    │ Aggregates     │                                   │
│                    │ CPU & Memory   │                                   │
│                    │ metrics        │                                   │
│                    └────────────────┘                                   │
│                             │                                           │
│                             ▼                                           │
│         ┌───────────────────┼───────────────────┐                       │
│         │                   │                   │                       │
│         ▼                   ▼                   ▼                       │
│   kubectl top          HPA              Scheduler                       │
│   (human view)    (auto-scaling)    (placement)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Install Metrics Server

```bash
# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify installation
kubectl get deployment metrics-server -n kube-system

# Check if working
kubectl top nodes
```

### 5.3 kubectl top

```bash
# Node resource usage
kubectl top nodes

# Output:
# NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1    250m         12%    1024Mi          25%
# node-2    180m         9%     890Mi           22%

# Pod resource usage
kubectl top pods

# Output:
# NAME       CPU(cores)   MEMORY(bytes)
# web-app    50m          128Mi
# db         200m         512Mi

# Pod usage in specific namespace
kubectl top pods -n kube-system

# Pod usage with containers
kubectl top pods --containers

# Sort by CPU or memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

---

## 6. Centralized Logging

### 6.1 Why Centralized Logging?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE LOGGING PROBLEM                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Centralized Logging:                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Pod dies → Logs LOST!                                        │   │
│   │   Node fails → Logs LOST!                                      │   │
│   │   Too many pods → Can't search!                                │   │
│   │   Multiple clusters → Chaos!                                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Centralized Logging:                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌─────┐ ┌─────┐ ┌─────┐                                      │   │
│   │   │Pod 1│ │Pod 2│ │Pod 3│   ... hundreds of pods              │   │
│   │   └──┬──┘ └──┬──┘ └──┬──┘                                      │   │
│   │      │       │       │                                          │   │
│   │      └───────┼───────┘                                          │   │
│   │              │                                                  │   │
│   │              ▼                                                  │   │
│   │      ┌──────────────┐                                          │   │
│   │      │  Log Shipper │  (Fluentd/Fluent Bit/Filebeat)          │   │
│   │      └──────────────┘                                          │   │
│   │              │                                                  │   │
│   │              ▼                                                  │   │
│   │      ┌──────────────┐                                          │   │
│   │      │  Log Storage │  (Elasticsearch/Loki/CloudWatch)        │   │
│   │      └──────────────┘                                          │   │
│   │              │                                                  │   │
│   │              ▼                                                  │   │
│   │      ┌──────────────┐                                          │   │
│   │      │    UI/Query  │  (Kibana/Grafana)                       │   │
│   │      └──────────────┘                                          │   │
│   │                                                                 │   │
│   │   ✓ Logs persist after pod death                               │   │
│   │   ✓ Search across all pods                                     │   │
│   │   ✓ Alerting on errors                                         │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Common Logging Stacks

| Stack | Components | Use Case |
|-------|------------|----------|
| **EFK** | Elasticsearch + Fluentd + Kibana | Traditional, full-featured |
| **ELK** | Elasticsearch + Logstash + Kibana | Same, different shipper |
| **PLG** | Promtail + Loki + Grafana | Lightweight, Prometheus-like |
| **Cloud** | CloudWatch, Stackdriver, Azure Monitor | Managed services |

### 6.3 Logging Patterns

| Pattern | Description |
|---------|-------------|
| **Node-level** | DaemonSet collects from `/var/log/containers/` |
| **Sidecar** | Sidecar container ships logs |
| **Direct** | App sends logs directly to backend |

---

## 7. Application Logging Best Practices

### 7.1 Log to stdout/stderr

```python
# Python - Good
import sys
print("Info message", file=sys.stdout)
print("Error message", file=sys.stderr)

# Don't write to files inside container
# Bad: with open('/var/log/app.log', 'w') as f: ...
```

### 7.2 Structured Logging (JSON)

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "message": "Database connection failed",
  "service": "api",
  "pod": "api-7d9f8c6b5-xyz",
  "trace_id": "abc123"
}
```

### 7.3 Sidecar Pattern for File Logs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Main application (writes to file)
  - name: app
    image: legacy-app
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  # Sidecar (streams file to stdout)
  - name: log-streamer
    image: busybox
    command: ['sh', '-c', 'tail -F /var/log/app/app.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  volumes:
  - name: logs
    emptyDir: {}
```

---

## 8. Monitoring with Prometheus

### 8.1 Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS MONITORING                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│   │  App Pod    │     │  App Pod    │     │  App Pod    │              │
│   │  /metrics   │     │  /metrics   │     │  /metrics   │              │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘              │
│          │                   │                   │                      │
│          │     PULL (scrape every 15s)          │                      │
│          └───────────────────┼───────────────────┘                      │
│                              │                                          │
│                              ▼                                          │
│                     ┌────────────────┐                                  │
│                     │   Prometheus   │                                  │
│                     │                │                                  │
│                     │  Scrapes       │                                  │
│                     │  Stores        │                                  │
│                     │  Queries       │                                  │
│                     │  Alerts        │                                  │
│                     └────────┬───────┘                                  │
│                              │                                          │
│              ┌───────────────┼───────────────┐                          │
│              │               │               │                          │
│              ▼               ▼               ▼                          │
│        ┌──────────┐   ┌──────────┐   ┌──────────────┐                  │
│        │ Grafana  │   │AlertMngr │   │   PromQL     │                  │
│        │ (UI)     │   │ (Alerts) │   │  (Queries)   │                  │
│        └──────────┘   └──────────┘   └──────────────┘                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 ServiceMonitor (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

---

## 9. Debugging with Logs

### 9.1 Common Debugging Scenarios

```bash
# Scenario 1: Pod won't start
kubectl describe pod <pod>           # Check events
kubectl logs <pod> --previous        # Check previous container logs

# Scenario 2: Application errors
kubectl logs <pod> | grep -i error
kubectl logs <pod> --since=1h | grep -i exception

# Scenario 3: Intermittent issues
kubectl logs <pod> -f --timestamps   # Watch with timestamps

# Scenario 4: Multi-container issues
kubectl logs <pod> --all-containers
kubectl logs <pod> -c init-container # Check init container
```

### 9.2 Debug Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEBUGGING FLOW                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1. Check Pod Status                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl get pod <pod>                                          │   │
│   │  kubectl describe pod <pod>                                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                        │                                                │
│                        ▼                                                │
│   2. Check Events                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl get events --field-selector involvedObject.name=<pod>  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                        │                                                │
│                        ▼                                                │
│   3. Check Container Logs                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl logs <pod>                                             │   │
│   │  kubectl logs <pod> --previous  (if restarting)                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                        │                                                │
│                        ▼                                                │
│   4. Check Resource Usage                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl top pod <pod>                                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                        │                                                │
│                        ▼                                                │
│   5. Interactive Debug                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl exec -it <pod> -- /bin/sh                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Command Reference

```bash
# LOGS
kubectl logs <pod>                       # Basic logs
kubectl logs <pod> -c <container>        # Specific container
kubectl logs <pod> --previous            # Previous instance
kubectl logs <pod> -f                    # Follow
kubectl logs <pod> --tail=100            # Last 100 lines
kubectl logs <pod> --since=1h            # Last hour
kubectl logs <pod> --timestamps          # With timestamps
kubectl logs -l app=nginx                # By label
kubectl logs deployment/web              # From deployment

# EVENTS
kubectl get events                       # All events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -w                    # Watch events

# METRICS
kubectl top nodes                        # Node metrics
kubectl top pods                         # Pod metrics
kubectl top pods -n <namespace>          # Namespace pods
kubectl top pods --containers            # Container level

# DESCRIBE (for debugging)
kubectl describe pod <pod>
kubectl describe node <node>
kubectl describe deployment <deploy>
```

---

## 11. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| kubectl logs | 🔴 HIGH | All options |
| Events | 🔴 HIGH | kubectl get events |
| kubectl top | 🟡 MEDIUM | Node and pod metrics |
| Component logs | 🟡 MEDIUM | journalctl, kubectl logs |

### Quick Reference for Exam

```bash
# Most common log commands
kubectl logs <pod>
kubectl logs <pod> -f --tail=50
kubectl logs <pod> --previous
kubectl logs -l app=myapp

# Events
kubectl get events --sort-by='.lastTimestamp'

# Metrics
kubectl top nodes
kubectl top pods
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Get logs from pod X, save to /tmp/logs.txt"             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl logs <pod-name> > /tmp/logs.txt                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Why is pod X not starting?"                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl describe pod <pod-name>   # Check Events section       │   │
│   │  kubectl logs <pod-name> --previous                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Which node is using most CPU?"                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl top nodes                                              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 4: "Get kubelet logs from node X"                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  ssh node-x                                                     │   │
│   │  journalctl -u kubelet                                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  LOGGING & MONITORING SUMMARY                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   LOGGING:                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Containers log to stdout/stderr                              │   │
│   │  • kubectl logs is your primary tool                            │   │
│   │  • Events show cluster-level activities                         │   │
│   │  • Centralized logging for production                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   MONITORING:                                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Metrics Server provides CPU/memory                           │   │
│   │  • kubectl top for quick resource check                         │   │
│   │  • Prometheus for production monitoring                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   KEY COMMANDS:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  kubectl logs <pod>                                             │   │
│   │  kubectl get events                                             │   │
│   │  kubectl top pods/nodes                                         │   │
│   │  kubectl describe <resource>                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Troubleshooting** - Systematic debugging of pods, nodes, and clusters
- Common failure patterns
- Debug containers
- Cluster troubleshooting

---

**Chapter 26 Complete! 🎉**

You now understand:
- Kubernetes logging architecture
- kubectl logs commands
- Kubernetes events
- Metrics server and kubectl top
- Centralized logging concepts
- Debugging with logs
- CKA exam preparation

