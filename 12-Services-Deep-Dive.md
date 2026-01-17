# ☸️ Chapter 12: Services Deep Dive

> Understanding Kubernetes networking with Services - the stable endpoint for your pods.

---

## 📚 Table of Contents

1. [What is a Service?](#-what-is-a-service)
2. [Why Services?](#-why-services)
3. [Service Types](#-service-types)
4. [ClusterIP Service](#-clusterip-service)
5. [NodePort Service](#-nodeport-service)
6. [LoadBalancer Service](#-loadbalancer-service)
7. [ExternalName Service](#-externalname-service)
8. [Service Discovery](#-service-discovery)
9. [DNS in Services](#-dns-in-services)
10. [Endpoints](#-endpoints)
11. [Session Affinity](#-session-affinity)
12. [Headless Services](#-headless-services)
13. [Service Without Selectors](#-service-without-selectors)
14. [Multi-Port Services](#-multi-port-services)
15. [Operations](#-operations)
16. [Troubleshooting](#-troubleshooting)
17. [CKA Exam Tips](#-cka-exam-tips)
18. [Summary](#-summary)

---

## 📖 What is a Service?

### Definition

> **Service** is an abstraction that defines a logical set of Pods and a policy to access them. Services provide stable networking for pods, which have dynamic IP addresses.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS A SERVICE?                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Service = Stable IP + DNS Name + Load Balancing                                    │
│                                                                                      │
│  Problem: Pods have dynamic IPs that change when pods restart                       │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │   Client                                                                    │   │
│  │     │                                                                       │   │
│  │     │  "Where is the nginx pod?"                                           │   │
│  │     │                                                                       │   │
│  │     ▼                                                                       │   │
│  │   ┌─────────────────────────────────────────────────────────────────────┐  │   │
│  │   │              Service: nginx-service                                 │  │   │
│  │   │              IP: 10.96.0.100 (STABLE!)                             │  │   │
│  │   │              DNS: nginx-service.default.svc.cluster.local          │  │   │
│  │   └─────────────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                              │   │
│  │              ┌───────────────┼───────────────┐                             │   │
│  │              │               │               │                             │   │
│  │              ▼               ▼               ▼                             │   │
│  │        ┌──────────┐   ┌──────────┐   ┌──────────┐                         │   │
│  │        │  Pod 1   │   │  Pod 2   │   │  Pod 3   │                         │   │
│  │        │10.244.1.5│   │10.244.2.8│   │10.244.3.2│   (dynamic IPs)         │   │
│  │        │  nginx   │   │  nginx   │   │  nginx   │                         │   │
│  │        └──────────┘   └──────────┘   └──────────┘                         │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Clients connect to Service IP → Service load balances to Pods                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Service Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Stable IP** | Service IP never changes |
| **DNS name** | Accessible by name |
| **Load balancing** | Distributes traffic to pods |
| **Service discovery** | Automatic endpoint updates |
| **Label selector** | Finds pods by labels |

---

## ❓ Why Services?

### The Problem

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         WHY WE NEED SERVICES                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  PROBLEM: Pod IPs are ephemeral                                                     │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Time T0:                          Time T1 (pod dies and restarts):                │
│  ┌──────────────┐                  ┌──────────────┐                                │
│  │   Pod        │                  │   Pod        │                                │
│  │   nginx      │                  │   nginx      │                                │
│  │   10.244.1.5 │     ────────►    │   10.244.2.9 │  ← DIFFERENT IP!               │
│  └──────────────┘                  └──────────────┘                                │
│                                                                                      │
│  If client was configured for 10.244.1.5, it can't find the pod anymore!           │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  SOLUTION: Service provides stable endpoint                                         │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │   Client → Service (10.96.0.100) → Pod (whatever IP it has now)             │ │
│  │                      ↑                                                        │ │
│  │              Always the same!                                                 │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Stable endpoint** | IP and DNS name never change |
| **Load balancing** | Traffic distributed across pods |
| **Service discovery** | Pods found automatically by labels |
| **Decoupling** | Clients don't need to know pod details |

---

## 📊 Service Types

### Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           SERVICE TYPES                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Type            │ Access From        │ Use Case                                    │
│  ────────────────┼────────────────────┼─────────────────────────────────────────────│
│  ClusterIP       │ Inside cluster     │ Internal services (default)                │
│  NodePort        │ External (node IP) │ Development, simple external access        │
│  LoadBalancer    │ External (LB IP)   │ Production external access                 │
│  ExternalName    │ DNS alias          │ Map to external service                    │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│                          Internet                                                    │
│                              │                                                       │
│                              ▼                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                      LoadBalancer (Cloud LB)                                  │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                       │
│                              ▼                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                      NodePort (Node:30000-32767)                              │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                       │
│                              ▼                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                      ClusterIP (10.96.x.x)                                    │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                       │
│                              ▼                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                           Pods                                                │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  LoadBalancer includes NodePort includes ClusterIP                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 ClusterIP Service

### Definition

> **ClusterIP** exposes the Service on a cluster-internal IP. The Service is only reachable from within the cluster. This is the default type.

### ClusterIP Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           CLUSTERIP SERVICE                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Kubernetes Cluster                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │   ┌────────────────────┐          ┌────────────────────────────────────────┐ │ │
│  │   │   Client Pod       │          │        ClusterIP Service              │ │ │
│  │   │                    │──────────│        nginx-service                  │ │ │
│  │   │   curl nginx-svc   │          │        10.96.0.100:80                 │ │ │
│  │   └────────────────────┘          └────────────────┬───────────────────────┘ │ │
│  │                                                    │                         │ │
│  │                                    ┌───────────────┼───────────────┐         │ │
│  │                                    │               │               │         │ │
│  │                                    ▼               ▼               ▼         │ │
│  │                              ┌──────────┐   ┌──────────┐   ┌──────────┐     │ │
│  │                              │  Pod 1   │   │  Pod 2   │   │  Pod 3   │     │ │
│  │                              │  nginx   │   │  nginx   │   │  nginx   │     │ │
│  │                              └──────────┘   └──────────┘   └──────────┘     │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  ❌ NOT accessible from outside the cluster                                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### ClusterIP Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP           # Default, can be omitted
  selector:
    app: nginx              # Pods with this label
  ports:
  - name: http
    port: 80                # Service port
    targetPort: 80          # Container port
    protocol: TCP
```

### Creating ClusterIP Service

```bash
# Imperative
kubectl expose deployment nginx --port=80 --type=ClusterIP

# Or
kubectl create service clusterip nginx --tcp=80:80

# Verify
kubectl get svc nginx-service
# NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.0.100    <none>        80/TCP    5s

# Test from within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- nginx-service
```

---

## 🌐 NodePort Service

### Definition

> **NodePort** exposes the Service on each Node's IP at a static port (NodePort). A ClusterIP Service, to which the NodePort Service routes, is automatically created.

### NodePort Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           NODEPORT SERVICE                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   External Traffic                                                                  │
│        │                                                                             │
│        │  curl 192.168.1.10:30080                                                   │
│        │                                                                             │
│        ▼                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │   Node 1 (192.168.1.10)    Node 2 (192.168.1.11)    Node 3 (192.168.1.12)    │ │
│  │         :30080                   :30080                   :30080              │ │
│  │           │                        │                        │                 │ │
│  │           └────────────────────────┼────────────────────────┘                 │ │
│  │                                    │                                          │ │
│  │                                    ▼                                          │ │
│  │                     ┌────────────────────────────┐                            │ │
│  │                     │    ClusterIP: 10.96.0.100  │                            │ │
│  │                     │    (auto-created)          │                            │ │
│  │                     └────────────────────────────┘                            │ │
│  │                                    │                                          │ │
│  │                    ┌───────────────┼───────────────┐                          │ │
│  │                    ▼               ▼               ▼                          │ │
│  │              ┌──────────┐   ┌──────────┐   ┌──────────┐                      │ │
│  │              │  Pod 1   │   │  Pod 2   │   │  Pod 3   │                      │ │
│  │              └──────────┘   └──────────┘   └──────────┘                      │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  NodePort range: 30000-32767                                                        │
│  Access via ANY node IP + NodePort                                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### NodePort Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - name: http
    port: 80              # ClusterIP port
    targetPort: 80        # Container port
    nodePort: 30080       # Node port (optional, auto-assigned if omitted)
    protocol: TCP
```

### Creating NodePort Service

```bash
# Imperative
kubectl expose deployment nginx --port=80 --type=NodePort

# With specific nodePort
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080

# Verify
kubectl get svc nginx-nodeport
# NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# nginx-nodeport   NodePort   10.96.0.101    <none>        80:30080/TCP   5s

# Access from outside cluster
curl http://<node-ip>:30080
```

---

## ⚖️ LoadBalancer Service

### Definition

> **LoadBalancer** exposes the Service externally using a cloud provider's load balancer. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created.

### LoadBalancer Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         LOADBALANCER SERVICE                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   Internet                                                                          │
│      │                                                                               │
│      │  curl http://a1b2c3-lb.us-east-1.elb.amazonaws.com                          │
│      │                                                                               │
│      ▼                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                    Cloud Load Balancer                                        │ │
│  │                    (AWS ELB / GCP LB / Azure LB)                             │ │
│  │                    External IP: 52.23.45.67                                  │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                       │
│                              ▼                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │   Node 1:30080            Node 2:30080            Node 3:30080               │ │
│  │      │                       │                       │                        │ │
│  │      └───────────────────────┼───────────────────────┘                        │ │
│  │                              │                                                │ │
│  │                              ▼                                                │ │
│  │                     ClusterIP: 10.96.0.100                                    │ │
│  │                              │                                                │ │
│  │              ┌───────────────┼───────────────┐                                │ │
│  │              ▼               ▼               ▼                                │ │
│  │        ┌──────────┐   ┌──────────┐   ┌──────────┐                            │ │
│  │        │  Pod 1   │   │  Pod 2   │   │  Pod 3   │                            │ │
│  │        └──────────┘   └──────────┘   └──────────┘                            │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  Requires cloud provider (AWS, GCP, Azure) or MetalLB for bare metal              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### LoadBalancer Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
  annotations:
    # AWS-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

### Creating LoadBalancer Service

```bash
# Imperative
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Verify
kubectl get svc nginx-loadbalancer
# NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
# nginx-loadbalancer   LoadBalancer   10.96.0.102    52.23.45.67      80:31234/TCP   1m

# Note: EXTERNAL-IP may show <pending> initially while LB is provisioning
```

---

## 🔗 ExternalName Service

### Definition

> **ExternalName** maps a Service to a DNS name, returning a CNAME record. No proxying is set up.

### Use Case

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNALNAME SERVICE                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Use case: Access external service by internal name                                 │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │  Kubernetes Cluster                                                           │ │
│  │                                                                               │ │
│  │   ┌──────────────┐        ┌─────────────────────────────────────────────────┐│ │
│  │   │   App Pod    │        │  ExternalName Service                          ││ │
│  │   │              │───────►│  Name: my-database                             ││ │
│  │   │ connect to   │        │  externalName: db.example.com                  ││ │
│  │   │ my-database  │        └──────────────────────────────────────────────────│ │
│  │   └──────────────┘                              │                             │ │
│  │                                                 │                             │ │
│  └─────────────────────────────────────────────────┼─────────────────────────────┘ │
│                                                    │                                │
│                                                    ▼                                │
│                               ┌─────────────────────────────────────────┐          │
│                               │     External Database                   │          │
│                               │     db.example.com                      │          │
│                               │     (outside cluster)                   │          │
│                               └─────────────────────────────────────────┘          │
│                                                                                      │
│  DNS lookup for my-database returns CNAME → db.example.com                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### ExternalName Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: db.example.com    # External DNS name
  # No selector, no ports, no clusterIP
```

---

## 🔍 Service Discovery

### Two Methods

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE DISCOVERY                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. DNS (Recommended)                                                               │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  CoreDNS provides DNS resolution for services:                                      │
│                                                                                      │
│  Service: nginx-service in namespace: default                                       │
│                                                                                      │
│  DNS Names:                                                                         │
│  • nginx-service                              (same namespace)                      │
│  • nginx-service.default                      (cross namespace)                     │
│  • nginx-service.default.svc                  (explicit)                            │
│  • nginx-service.default.svc.cluster.local    (FQDN)                               │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  2. Environment Variables (Legacy)                                                  │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  When pod starts, K8s injects env vars for each service:                           │
│                                                                                      │
│  NGINX_SERVICE_SERVICE_HOST=10.96.0.100                                            │
│  NGINX_SERVICE_SERVICE_PORT=80                                                      │
│  NGINX_SERVICE_PORT=tcp://10.96.0.100:80                                           │
│                                                                                      │
│  ⚠️  Limitation: Only services created BEFORE the pod are injected                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📛 DNS in Services

### DNS Resolution

```bash
# Inside a pod, access service by name
curl http://nginx-service           # Same namespace
curl http://nginx-service.default   # Cross namespace
curl http://nginx-service.default.svc.cluster.local  # FQDN

# DNS lookup
nslookup nginx-service
# Server:    10.96.0.10 (CoreDNS)
# Address:   10.96.0.10#53
# Name:      nginx-service.default.svc.cluster.local
# Address:   10.96.0.100
```

### DNS Record Types

| Service Type | DNS Record |
|--------------|------------|
| ClusterIP | A record → ClusterIP |
| Headless | A record → Pod IPs |
| ExternalName | CNAME → external name |

---

## 🎯 Endpoints

### What are Endpoints?

> **Endpoints** are the actual IP addresses of pods backing a Service. Kubernetes automatically maintains Endpoints for Services with selectors.

```yaml
# View endpoints
kubectl get endpoints nginx-service
# NAME            ENDPOINTS                                      AGE
# nginx-service   10.244.1.5:80,10.244.2.8:80,10.244.3.2:80     5m
```

### Endpoints Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         ENDPOINTS FLOW                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. Service with selector                                                           │
│     ┌─────────────────────────────┐                                                 │
│     │ Service: nginx-service      │                                                 │
│     │ selector:                   │                                                 │
│     │   app: nginx                │                                                 │
│     └─────────────────────────────┘                                                 │
│                                                                                      │
│  2. Endpoint Controller watches pods with matching labels                           │
│                                                                                      │
│  3. Endpoints object auto-created and updated                                       │
│     ┌─────────────────────────────┐                                                 │
│     │ Endpoints: nginx-service    │                                                 │
│     │ subsets:                    │                                                 │
│     │ - addresses:                │                                                 │
│     │   - ip: 10.244.1.5          │  ← Pod IPs                                     │
│     │   - ip: 10.244.2.8          │                                                 │
│     │   - ip: 10.244.3.2          │                                                 │
│     │   ports:                    │                                                 │
│     │   - port: 80                │                                                 │
│     └─────────────────────────────┘                                                 │
│                                                                                      │
│  4. kube-proxy uses Endpoints to configure iptables/IPVS rules                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Session Affinity

### Definition

> **Session Affinity** ensures that requests from the same client are directed to the same Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  sessionAffinity: ClientIP       # Default: None
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800       # 3 hours (default)
  ports:
  - port: 80
    targetPort: 80
```

### Session Affinity Types

| Type | Behavior |
|------|----------|
| `None` | Random pod selection (default) |
| `ClientIP` | Same client IP → same pod |

---

## 🔊 Headless Services

### Definition

> **Headless Service** has `clusterIP: None`. Instead of load balancing, DNS returns the IPs of all pods directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None              # Makes it headless!
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### Headless vs Regular

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         HEADLESS VS REGULAR SERVICE                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Regular Service:                                                                   │
│  DNS: nginx-service → 10.96.0.100 (ClusterIP)                                      │
│                                                                                      │
│  Headless Service:                                                                  │
│  DNS: nginx-headless → 10.244.1.5, 10.244.2.8, 10.244.3.2 (Pod IPs)               │
│                                                                                      │
│  Use cases for headless:                                                            │
│  • StatefulSets (need individual pod access)                                       │
│  • Client-side load balancing                                                      │
│  • Service discovery without proxy                                                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Service Without Selectors

### Use Case: External Services

```yaml
# Service without selector
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 3306
    targetPort: 3306
  # No selector!

---

# Manually create Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db     # Must match Service name
subsets:
- addresses:
  - ip: 192.168.1.100   # External database IP
  ports:
  - port: 3306
```

---

## 🔢 Multi-Port Services

### Definition

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - name: http            # Name required for multi-port
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

---

## 🔧 Operations

### Common Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

kubectl create service clusterip nginx --tcp=80:80
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get services
kubectl get svc                      # Short form
kubectl get svc -o wide
kubectl get svc nginx -o yaml

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe svc nginx

# ═══════════════════════════════════════════════════════════════════
# ENDPOINTS
# ═══════════════════════════════════════════════════════════════════
kubectl get endpoints
kubectl get ep nginx

# ═══════════════════════════════════════════════════════════════════
# TEST
# ═══════════════════════════════════════════════════════════════════
# From within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- http://nginx-service

# Port forward
kubectl port-forward svc/nginx 8080:80

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete svc nginx
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Service not reachable | No endpoints | Check selector matches pod labels |
| Endpoints empty | Pods not ready | Check pod readiness |
| DNS not resolving | CoreDNS issue | Check CoreDNS pods |
| External traffic fails | Wrong service type | Use NodePort or LoadBalancer |

### Debug Commands

```bash
# Check service
kubectl describe svc nginx

# Check endpoints
kubectl get endpoints nginx

# Check if pods match selector
kubectl get pods -l app=nginx

# Test DNS resolution
kubectl run test --image=busybox --rm -it -- nslookup nginx-service

# Test connectivity
kubectl run test --image=busybox --rm -it -- wget -qO- http://nginx-service

# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

---

## 🎓 CKA Exam Tips

### Quick Service Creation

```bash
# Expose deployment
kubectl expose deployment nginx --port=80 --type=NodePort

# Generate YAML
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml > svc.yaml

# Create service directly
kubectl create service clusterip nginx --tcp=80:80
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080
```

### Key Points

1. **Default type is ClusterIP**
2. **NodePort range: 30000-32767**
3. **Selector must match pod labels**
4. **Endpoints auto-created for services with selectors**
5. **Headless: clusterIP: None**

---

## ✅ Summary

### Service Types

| Type | Access | Use Case |
|------|--------|----------|
| ClusterIP | Internal only | Default, internal services |
| NodePort | Node IP:Port | Development, simple external |
| LoadBalancer | External LB IP | Production external |
| ExternalName | DNS alias | External services |

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Service** | Stable endpoint for pods |
| **Selector** | Finds pods by labels |
| **Endpoints** | Pod IPs backing service |
| **ClusterIP** | Virtual IP for service |
| **Headless** | No ClusterIP, returns pod IPs |

---

## 🔜 What's Next

In **Chapter 13: Ingress**, we'll cover:

- HTTP/HTTPS routing
- Path-based routing
- Host-based routing
- TLS termination
- Ingress controllers

---

*Services are the networking foundation - master them before moving to Ingress!*

