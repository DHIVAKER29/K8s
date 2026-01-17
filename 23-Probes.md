# Chapter 23: Probes - Container Health Checks

## Introduction

How does Kubernetes know if your application is healthy? How does it know when to restart a container, or when to send traffic? **Probes** answer these questions by checking container health at regular intervals.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE HEALTH CHECK PROBLEM                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Probes:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌────────────────────────────────────────────────────────┐    │   │
│   │   │                     POD                                │    │   │
│   │   │   ┌────────────────────────────────────────────────┐   │    │   │
│   │   │   │              CONTAINER                         │   │    │   │
│   │   │   │                                                │   │    │   │
│   │   │   │   Process Running ✓                            │   │    │   │
│   │   │   │   But: Application Deadlocked! ❌              │   │    │   │
│   │   │   │       Database Connection Lost! ❌             │   │    │   │
│   │   │   │       Serving 500 Errors! ❌                   │   │    │   │
│   │   │   │                                                │   │    │   │
│   │   │   └────────────────────────────────────────────────┘   │    │   │
│   │   └────────────────────────────────────────────────────────┘    │   │
│   │                                                                 │   │
│   │   Kubernetes: "Container is running, all good!" 👍             │   │
│   │   Reality: Application is broken! 💀                           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Probes:                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Kubelet periodically checks:                                  │   │
│   │   ├── Liveness: "Is app alive?" → Restart if dead             │   │
│   │   ├── Readiness: "Is app ready?" → Remove from service        │   │
│   │   └── Startup: "Has app started?" → Wait for slow apps        │   │
│   │                                                                 │   │
│   │   Kubernetes: "App is unhealthy, restarting!" ✓                │   │
│   │   Reality: Self-healing in action! 🔄                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. The Three Probe Types

### 1.1 Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      THREE PROBE TYPES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    STARTUP PROBE                                │   │
│   │                                                                 │   │
│   │   Question: "Has the container finished starting up?"           │   │
│   │                                                                 │   │
│   │   • Runs FIRST, before other probes                            │   │
│   │   • Disables liveness/readiness until it succeeds              │   │
│   │   • For slow-starting applications                             │   │
│   │   • On failure: Container is killed and restarted              │   │
│   │                                                                 │   │
│   │   Use for: Java apps, legacy apps, apps with long init         │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │ Once successful                         │
│                               ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    LIVENESS PROBE                               │   │
│   │                                                                 │   │
│   │   Question: "Is the container still alive and healthy?"         │   │
│   │                                                                 │   │
│   │   • Runs continuously throughout container lifetime            │   │
│   │   • Detects deadlocks, hangs, infinite loops                   │   │
│   │   • On failure: Container is killed and restarted              │   │
│   │                                                                 │   │
│   │   Use for: All containers that can get stuck                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                               │ Runs in parallel                        │
│                               ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                   READINESS PROBE                               │   │
│   │                                                                 │   │
│   │   Question: "Is the container ready to receive traffic?"        │   │
│   │                                                                 │   │
│   │   • Controls Service endpoint membership                       │   │
│   │   • Does NOT restart container on failure                      │   │
│   │   • On failure: Pod removed from Service endpoints             │   │
│   │   • On success: Pod added back to Service endpoints            │   │
│   │                                                                 │   │
│   │   Use for: Load balancing, dependency checks                   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Probe Comparison

| Aspect | Liveness | Readiness | Startup |
|--------|----------|-----------|---------|
| **Purpose** | Is container alive? | Ready for traffic? | Finished starting? |
| **On Failure** | Restart container | Remove from Service | Restart container |
| **When Runs** | After startup | After startup | Before liveness/readiness |
| **Typical Use** | Detect deadlocks | Load balancing | Slow-starting apps |
| **Blocks Traffic** | No | Yes | Yes (indirectly) |

---

## 2. Probe Mechanisms

### 2.1 Three Ways to Check Health

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PROBE MECHANISMS                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   HTTP GET                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Kubelet ──▶ GET /healthz ──▶ Container                       │   │
│   │              ◀── HTTP 200-399 ◀──                              │   │
│   │                                                                 │   │
│   │   Success: Status code 200-399                                  │   │
│   │   Failure: Status code >= 400 or no response                   │   │
│   │                                                                 │   │
│   │   Best for: Web applications, APIs                             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   TCP Socket                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Kubelet ──▶ TCP Connect :3306 ──▶ Container                  │   │
│   │              ◀── Connection Established ◀──                    │   │
│   │                                                                 │   │
│   │   Success: TCP connection established                          │   │
│   │   Failure: Connection refused or timeout                       │   │
│   │                                                                 │   │
│   │   Best for: Databases, TCP services without HTTP               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Exec Command                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Kubelet ──▶ Exec "cat /tmp/healthy" ──▶ Container            │   │
│   │              ◀── Exit code 0 ◀──                               │   │
│   │                                                                 │   │
│   │   Success: Command exits with code 0                           │   │
│   │   Failure: Command exits with non-zero code                    │   │
│   │                                                                 │   │
│   │   Best for: Custom health scripts, non-HTTP apps               │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   gRPC (Kubernetes 1.24+)                                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Kubelet ──▶ gRPC Health Check ──▶ Container                  │   │
│   │              ◀── SERVING status ◀──                            │   │
│   │                                                                 │   │
│   │   Success: gRPC health check returns SERVING                   │   │
│   │   Failure: Returns NOT_SERVING or error                        │   │
│   │                                                                 │   │
│   │   Best for: gRPC services                                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Liveness Probe

### 3.1 When to Use

Use liveness probes when your application can get into a broken state that requires a restart:
- Deadlocks
- Infinite loops
- Memory corruption
- Unresponsive due to bugs

### 3.2 HTTP Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: app
    image: myapp
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz            # Endpoint to check
        port: 8080                # Port to check
        httpHeaders:              # Optional custom headers
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15     # Wait before first probe
      periodSeconds: 10           # How often to probe
      timeoutSeconds: 3           # Timeout for probe
      successThreshold: 1         # Successes to be healthy
      failureThreshold: 3         # Failures before restart
```

### 3.3 TCP Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
spec:
  containers:
  - name: db
    image: postgres
    ports:
    - containerPort: 5432
    livenessProbe:
      tcpSocket:
        port: 5432
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
```

### 3.4 Exec Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'touch /tmp/healthy && sleep 3600']
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

---

## 4. Readiness Probe

### 4.1 When to Use

Use readiness probes when your application:
- Needs time to load data before serving
- Depends on external services (database, cache)
- Should temporarily stop receiving traffic
- Has maintenance windows

### 4.2 Readiness Probe Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    READINESS PROBE FLOW                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      SERVICE                                    │   │
│   │                                                                 │   │
│   │   Endpoints: [10.0.0.1, 10.0.0.2, 10.0.0.3]                    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                           │                                             │
│                           │ Load balances to                            │
│                           ▼                                             │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│   │  Pod 1 (Ready)   │  │  Pod 2 (Ready)   │  │ Pod 3 (NotReady) │     │
│   │  10.0.0.1 ✓      │  │  10.0.0.2 ✓      │  │ 10.0.0.3 ❌      │     │
│   │                  │  │                  │  │                  │     │
│   │  Gets traffic    │  │  Gets traffic    │  │  NO traffic      │     │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│                                                                         │
│   When Pod 3's readiness probe passes:                                  │
│   • Pod 3 is added back to Service endpoints                           │
│   • Traffic resumes to Pod 3                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 HTTP Readiness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-http
spec:
  containers:
  - name: app
    image: myapp
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /ready              # Different from liveness!
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
```

### 4.4 Readiness with Dependency Check

```yaml
# Your app's /ready endpoint checks:
# - Database connection
# - Cache connection
# - Required files loaded
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

---

## 5. Startup Probe

### 5.1 When to Use

Use startup probes for:
- Applications with long startup times
- Legacy applications
- Applications with unpredictable startup
- Preventing false-positive liveness failures during startup

### 5.2 The Startup Problem

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE STARTUP PROBLEM                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Startup Probe (Problem):                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Timeline:                                                     │   │
│   │   0s        30s       60s       90s       120s                 │   │
│   │   |---------|---------|---------|---------|                    │   │
│   │   │ Container starting...                 │                    │   │
│   │   │ Loading data... Still loading...      │                    │   │
│   │   │                                       │                    │   │
│   │   │     ❌ Liveness fails! (30s)          │                    │   │
│   │   │     Container restarted!              │                    │   │
│   │   │                                       │                    │   │
│   │   │         ❌ Liveness fails again! (60s)│                    │   │
│   │   │         Container restarted!          │                    │   │
│   │   │                                       │                    │   │
│   │   │ Result: Restart loop! Never starts!  │                    │   │
│   │   │                                       │                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Startup Probe (Solution):                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Timeline:                                                     │   │
│   │   0s        30s       60s       90s       120s                 │   │
│   │   |---------|---------|---------|---------|                    │   │
│   │   │ Startup probe running...              │                    │   │
│   │   │ Liveness/Readiness DISABLED           │                    │   │
│   │   │                                       │                    │   │
│   │   │                      ✓ Startup passes (90s)                │   │
│   │   │                      Liveness/Readiness enabled            │   │
│   │   │                                       │                    │   │
│   │   │ Result: Container starts successfully!│                    │   │
│   │   │                                       │                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Startup Probe Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-starting-app
spec:
  containers:
  - name: app
    image: legacy-java-app
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 30        # 30 * 10s = 300s (5 min) to start
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

---

## 6. Probe Configuration Parameters

### 6.1 Parameters Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initialDelaySeconds` | 0 | Seconds to wait before first probe |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Seconds before probe times out |
| `successThreshold` | 1 | Consecutive successes to be healthy |
| `failureThreshold` | 3 | Consecutive failures before action |

### 6.2 Calculating Maximum Startup Time

```
Maximum startup time = failureThreshold × periodSeconds

Example:
  failureThreshold: 30
  periodSeconds: 10
  Maximum startup time = 30 × 10 = 300 seconds (5 minutes)
```

### 6.3 Configuration Guidelines

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  PROBE CONFIGURATION GUIDELINES                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   initialDelaySeconds                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Set to average startup time                                  │   │
│   │  • Too low: Unnecessary probe failures during startup           │   │
│   │  • Too high: Delayed detection of issues                        │   │
│   │  • With startupProbe: Can be 0 for liveness/readiness          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   periodSeconds                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Balance between responsiveness and overhead                  │   │
│   │  • Liveness: 10-30 seconds typical                             │   │
│   │  • Readiness: 5-10 seconds typical (faster response)           │   │
│   │  • Too frequent: CPU overhead, log noise                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   timeoutSeconds                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • How long to wait for response                                │   │
│   │  • Should be less than periodSeconds                           │   │
│   │  • Increase for slow endpoints                                 │   │
│   │  • Default (1s) often too aggressive                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   failureThreshold                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • How many failures before action                              │   │
│   │  • Higher = More tolerance for transient issues                │   │
│   │  • Lower = Faster recovery from real failures                  │   │
│   │  • 3 is a good default                                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Complete Examples

### 7.1 Web Application with All Three Probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: myapp:latest
        ports:
        - containerPort: 8080
        
        # Startup probe for slow starts
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 60       # 5 minutes max startup
        
        # Liveness probe - is it alive?
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0     # Starts after startup probe
          periodSeconds: 15
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Readiness probe - ready for traffic?
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 0     # Starts after startup probe
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
```

### 7.2 Database Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:15
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_PASSWORD
      value: secret
    
    livenessProbe:
      exec:
        command:
        - pg_isready
        - -U
        - postgres
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    
    readinessProbe:
      exec:
        command:
        - pg_isready
        - -U
        - postgres
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 7.3 gRPC Service (K8s 1.24+)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grpc-service
spec:
  containers:
  - name: grpc
    image: my-grpc-app
    ports:
    - containerPort: 50051
    
    livenessProbe:
      grpc:
        port: 50051
      initialDelaySeconds: 10
      periodSeconds: 10
    
    readinessProbe:
      grpc:
        port: 50051
        service: my-service      # Optional service name
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

## 8. Best Practices

### 8.1 Probe Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PROBE BEST PRACTICES                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Use separate endpoints for liveness and readiness           │   │
│   │    - /healthz for liveness (basic alive check)                 │   │
│   │    - /ready for readiness (dependencies check)                 │   │
│   │                                                                 │   │
│   │  • Keep liveness probes SIMPLE and FAST                        │   │
│   │    - Don't check dependencies in liveness                      │   │
│   │    - Dependency failure shouldn't restart your app             │   │
│   │                                                                 │   │
│   │  • Use startup probes for slow-starting apps                   │   │
│   │    - Java apps, apps loading large datasets                    │   │
│   │                                                                 │   │
│   │  • Set appropriate timeouts                                    │   │
│   │    - timeoutSeconds should match endpoint response time        │   │
│   │                                                                 │   │
│   │  • Test probes in staging first                                │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │  • Check database in liveness probe                            │   │
│   │    - DB down → App restarts → More DB load → Worse!           │   │
│   │                                                                 │   │
│   │  • Use same endpoint for liveness and readiness               │   │
│   │    - They have different purposes                              │   │
│   │                                                                 │   │
│   │  • Set failureThreshold too low (1)                           │   │
│   │    - Single transient failure causes restart                   │   │
│   │                                                                 │   │
│   │  • Set initialDelaySeconds without understanding startup       │   │
│   │    - Too low: Premature failures                              │   │
│   │    - Too high: Slow issue detection                           │   │
│   │                                                                 │   │
│   │  • Make probes do heavy work                                  │   │
│   │    - Probes run frequently; keep them lightweight              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Liveness vs Readiness Design

| Check | Liveness | Readiness |
|-------|----------|-----------|
| App process running | ✅ | ✅ |
| Can handle HTTP | ✅ | ✅ |
| Database connected | ❌ | ✅ |
| Cache connected | ❌ | ✅ |
| Disk space available | ❌ | ✅ |
| Dependencies healthy | ❌ | ✅ |

---

## 9. Troubleshooting

### 9.1 Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Pod stuck in `CrashLoopBackOff` | Liveness probe failing too early | Increase `initialDelaySeconds` or use startup probe |
| Pod never becomes Ready | Readiness probe failing | Check probe endpoint, increase timeout |
| Pods restarting randomly | Liveness probe too aggressive | Increase `failureThreshold` or `timeoutSeconds` |
| Slow rolling updates | Readiness probe slow | Decrease `periodSeconds` |

### 9.2 Debug Commands

```bash
# Check pod events for probe failures
kubectl describe pod <pod-name> | grep -A 10 Events

# Check probe status
kubectl get pod <pod-name> -o yaml | grep -A 20 livenessProbe

# Test endpoint manually
kubectl exec <pod-name> -- curl -v http://localhost:8080/healthz

# Watch pod status
kubectl get pod <pod-name> -w
```

### 9.3 Sample Event Output

```
Events:
  Type     Reason     Age   Message
  ----     ------     ----  -------
  Warning  Unhealthy  30s   Liveness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy  20s   Liveness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy  10s   Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    10s   Container app failed liveness probe, will be restarted
```

---

## 10. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Liveness probe | 🔴 HIGH | HTTP, TCP, exec |
| Readiness probe | 🔴 HIGH | HTTP, TCP, exec |
| Probe parameters | 🟡 MEDIUM | initialDelaySeconds, periodSeconds |
| Startup probe | 🟡 MEDIUM | Slow-starting apps |

### Quick Reference for Exam

```yaml
# HTTP probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

# TCP probe
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10

# Exec probe
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Add liveness probe checking /healthz on port 8080"       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  livenessProbe:                                                 │   │
│   │    httpGet:                                                     │   │
│   │      path: /healthz                                             │   │
│   │      port: 8080                                                 │   │
│   │    initialDelaySeconds: 15                                      │   │
│   │    periodSeconds: 10                                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Add readiness probe with exec command"                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  readinessProbe:                                                │   │
│   │    exec:                                                        │   │
│   │      command:                                                   │   │
│   │      - cat                                                      │   │
│   │      - /tmp/ready                                               │   │
│   │    initialDelaySeconds: 5                                       │   │
│   │    periodSeconds: 5                                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Pod keeps restarting - fix the probe"                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Check: kubectl describe pod <pod>                              │   │
│   │  Look for: "Liveness probe failed"                              │   │
│   │  Fix: Increase initialDelaySeconds or add startupProbe         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       PROBES SUMMARY                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Container Starts                                                      │
│        │                                                                │
│        ▼                                                                │
│   ┌─────────────────┐                                                   │
│   │  Startup Probe  │  "Has it started?"                               │
│   │  (if defined)   │  Failure → Restart                               │
│   └────────┬────────┘                                                   │
│            │ Success                                                    │
│            ▼                                                            │
│   ┌─────────────────────────────────────────────────────────┐          │
│   │                                                         │          │
│   │  ┌─────────────────┐      ┌─────────────────┐          │          │
│   │  │  Liveness Probe │      │ Readiness Probe │          │          │
│   │  │                 │      │                 │          │          │
│   │  │ "Is it alive?"  │      │ "Is it ready?"  │          │          │
│   │  │                 │      │                 │          │          │
│   │  │ Failure →       │      │ Failure →       │          │          │
│   │  │ Restart         │      │ No traffic      │          │          │
│   │  │                 │      │                 │          │          │
│   │  └─────────────────┘      └─────────────────┘          │          │
│   │                                                         │          │
│   │               Both run continuously                     │          │
│   └─────────────────────────────────────────────────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Horizontal Pod Autoscaler (HPA)** - Automatic scaling based on metrics
- CPU and memory-based scaling
- Custom metrics scaling
- Scaling behavior configuration

---

**Chapter 23 Complete! 🎉**

You now understand:
- Liveness probes (is it alive?)
- Readiness probes (is it ready for traffic?)
- Startup probes (has it finished starting?)
- Probe mechanisms (HTTP, TCP, exec, gRPC)
- Configuration parameters
- Best practices
- Troubleshooting
- CKA exam preparation

