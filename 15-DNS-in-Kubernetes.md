# ☸️ Chapter 15: DNS in Kubernetes

> Understanding service discovery and name resolution in Kubernetes with CoreDNS.

---

## 📚 Table of Contents

1. [What is Kubernetes DNS?](#-what-is-kubernetes-dns)
2. [CoreDNS Architecture](#-coredns-architecture)
3. [Service DNS Records](#-service-dns-records)
4. [Pod DNS Records](#-pod-dns-records)
5. [DNS Resolution Flow](#-dns-resolution-flow)
6. [DNS Policies](#-dns-policies)
7. [Custom DNS Configuration](#-custom-dns-configuration)
8. [Headless Services DNS](#-headless-services-dns)
9. [External DNS](#-external-dns)
10. [CoreDNS Configuration](#-coredns-configuration)
11. [Debugging DNS](#-debugging-dns)
12. [Common Issues](#-common-issues)
13. [CKA Exam Tips](#-cka-exam-tips)
14. [Summary](#-summary)

---

## 📖 What is Kubernetes DNS?

### Definition

> **Kubernetes DNS** is a cluster add-on that provides DNS-based service discovery. It allows pods to discover and communicate with services using DNS names instead of IP addresses.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES DNS                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Problem: Services have dynamic IPs, how do pods find them?                         │
│                                                                                      │
│  Solution: DNS! Every service gets a DNS name automatically.                        │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                             │   │
│  │   Pod                          CoreDNS                    Service          │   │
│  │   ┌──────────┐                ┌──────────┐              ┌──────────┐       │   │
│  │   │          │   "Where is    │          │              │ nginx    │       │   │
│  │   │  app     │   nginx-svc?"  │  DNS     │              │ service  │       │   │
│  │   │          │ ──────────────►│  Server  │              │          │       │   │
│  │   │          │                │          │              │ 10.96.0.5│       │   │
│  │   │          │  "10.96.0.5"   │          │              │          │       │   │
│  │   │          │ ◄──────────────│          │              │          │       │   │
│  │   │          │                └──────────┘              └──────────┘       │   │
│  │   │          │                                                │            │   │
│  │   │          │ ─────────────────────────────────────────────►│            │   │
│  │   │          │            Connect to 10.96.0.5               │            │   │
│  │   └──────────┘                                                             │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  DNS Names:                                                                         │
│  • nginx-svc                         (same namespace)                               │
│  • nginx-svc.default                 (cross namespace)                              │
│  • nginx-svc.default.svc.cluster.local (FQDN)                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ CoreDNS Architecture

### CoreDNS Components

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         COREDNS ARCHITECTURE                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                          kube-system namespace                              │   │
│  │                                                                             │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│  │  │                      CoreDNS Deployment                             │   │   │
│  │  │                                                                     │   │   │
│  │  │   ┌─────────────┐        ┌─────────────┐                           │   │   │
│  │  │   │ coredns-xxx │        │ coredns-yyy │                           │   │   │
│  │  │   │  (replica)  │        │  (replica)  │                           │   │   │
│  │  │   └─────────────┘        └─────────────┘                           │   │   │
│  │  │                                                                     │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                              │   │
│  │                              ▼                                              │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│  │  │                    kube-dns Service                                 │   │   │
│  │  │                    ClusterIP: 10.96.0.10                            │   │   │
│  │  │                    Port: 53 (DNS)                                   │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                             │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│  │  │                    coredns ConfigMap                                │   │   │
│  │  │                    (Corefile configuration)                         │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Every pod's /etc/resolv.conf points to 10.96.0.10 (kube-dns service)             │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### View CoreDNS Components

```bash
# CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS service
kubectl get svc -n kube-system kube-dns

# CoreDNS ConfigMap
kubectl get configmap -n kube-system coredns -o yaml

# CoreDNS deployment
kubectl get deployment -n kube-system coredns
```

---

## 🌐 Service DNS Records

### DNS Name Format

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE DNS FORMAT                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Full format:                                                                       │
│  <service-name>.<namespace>.svc.<cluster-domain>                                   │
│                                                                                      │
│  Example:                                                                           │
│  nginx-service.default.svc.cluster.local                                           │
│       │           │     │      │                                                    │
│       │           │     │      └── Cluster domain (usually cluster.local)          │
│       │           │     └── Always "svc" for services                              │
│       │           └── Namespace name                                                │
│       └── Service name                                                              │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Shorthand options (from pods):                                                     │
│                                                                                      │
│  Same namespace:                                                                    │
│  • nginx-service                                                                    │
│                                                                                      │
│  Cross namespace:                                                                   │
│  • nginx-service.production                                                         │
│  • nginx-service.production.svc                                                    │
│  • nginx-service.production.svc.cluster.local                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### DNS Record Types

| Service Type | DNS Record | Returns |
|--------------|------------|---------|
| ClusterIP | A record | ClusterIP address |
| Headless | A record | Pod IP addresses |
| ExternalName | CNAME | External DNS name |

### Service DNS Examples

```bash
# Create a service
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80

# Test DNS resolution from a pod
kubectl run test --image=busybox:1.28 --rm -it -- nslookup nginx
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# 
# Name:      nginx
# Address 1: 10.96.45.123 nginx.default.svc.cluster.local

# Test with FQDN
kubectl run test --image=busybox:1.28 --rm -it -- nslookup nginx.default.svc.cluster.local
```

---

## 🔵 Pod DNS Records

### Pod DNS Format

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         POD DNS FORMAT                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Pod A Record (IP-based):                                                           │
│  <pod-ip-dashed>.<namespace>.pod.<cluster-domain>                                  │
│                                                                                      │
│  Example:                                                                           │
│  Pod IP: 10.244.1.5                                                                 │
│  DNS: 10-244-1-5.default.pod.cluster.local                                         │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  With Headless Service (StatefulSet):                                              │
│  <pod-name>.<service-name>.<namespace>.svc.<cluster-domain>                        │
│                                                                                      │
│  Example:                                                                           │
│  mysql-0.mysql-headless.default.svc.cluster.local                                  │
│  mysql-1.mysql-headless.default.svc.cluster.local                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Pod DNS Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  containers:
  - name: app
    image: nginx
  
  # Custom hostname and subdomain
  hostname: my-host              # Pod's hostname
  subdomain: my-subdomain        # Creates DNS: my-host.my-subdomain.<ns>.svc.<domain>
  
  # DNS configuration
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - my-domain.local
    options:
    - name: ndots
      value: "5"
```

---

## 🔄 DNS Resolution Flow

### How DNS Resolution Works

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         DNS RESOLUTION FLOW                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. Pod makes DNS query                                                             │
│                                                                                      │
│  ┌──────────────────┐                                                               │
│  │ Pod              │                                                               │
│  │                  │                                                               │
│  │ curl nginx-svc   │                                                               │
│  │      │           │                                                               │
│  │      ▼           │                                                               │
│  │ /etc/resolv.conf │                                                               │
│  │ nameserver 10.96.0.10                                                            │
│  │ search default.svc.cluster.local svc.cluster.local cluster.local                │
│  └──────────────────┘                                                               │
│           │                                                                          │
│           │ Query: nginx-svc                                                        │
│           │                                                                          │
│           ▼                                                                          │
│  2. CoreDNS receives query                                                          │
│                                                                                      │
│  ┌──────────────────┐                                                               │
│  │ CoreDNS          │                                                               │
│  │ (10.96.0.10:53)  │                                                               │
│  │                  │                                                               │
│  │ Try: nginx-svc.default.svc.cluster.local                                        │
│  │                  │                                                               │
│  │ Found! → 10.96.45.123                                                           │
│  └──────────────────┘                                                               │
│           │                                                                          │
│           │ Response: 10.96.45.123                                                  │
│           ▼                                                                          │
│  3. Pod receives IP and connects                                                    │
│                                                                                      │
│  ┌──────────────────┐          ┌──────────────────┐                                │
│  │ Pod              │──────────│ Service          │                                │
│  │                  │          │ nginx-svc        │                                │
│  │ Connect to       │          │ 10.96.45.123     │                                │
│  │ 10.96.45.123     │          │                  │                                │
│  └──────────────────┘          └──────────────────┘                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Pod's /etc/resolv.conf

```bash
# View a pod's DNS configuration
kubectl exec -it nginx-pod -- cat /etc/resolv.conf

# Output:
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

| Field | Description |
|-------|-------------|
| `nameserver` | CoreDNS service IP |
| `search` | Domains to append for short names |
| `ndots:5` | If name has < 5 dots, try search domains first |

---

## 📋 DNS Policies

### dnsPolicy Options

```yaml
spec:
  dnsPolicy: ClusterFirst    # Default
```

| Policy | Behavior |
|--------|----------|
| `ClusterFirst` | Use cluster DNS, fallback to upstream (default) |
| `ClusterFirstWithHostNet` | Use cluster DNS for hostNetwork pods |
| `Default` | Inherit from node's DNS |
| `None` | Ignore all DNS settings, use dnsConfig only |

### Examples

```yaml
# Default: ClusterFirst
apiVersion: v1
kind: Pod
metadata:
  name: default-dns
spec:
  containers:
  - name: app
    image: nginx
  dnsPolicy: ClusterFirst

---

# HostNetwork with ClusterDNS
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-pod
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: app
    image: nginx

---

# Custom DNS only
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-only
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - my-company.local
  containers:
  - name: app
    image: nginx
```

---

## ⚙️ Custom DNS Configuration

### dnsConfig Options

```yaml
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    # Additional nameservers
    nameservers:
    - 1.2.3.4
    
    # Additional search domains
    searches:
    - my-company.local
    - svc.cluster.local
    
    # DNS options
    options:
    - name: ndots
      value: "2"
    - name: edns0
```

### Use Cases

```yaml
# Add corporate DNS for internal services
apiVersion: v1
kind: Pod
metadata:
  name: corp-pod
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
    - 10.0.0.53           # Corporate DNS
    searches:
    - corp.internal       # Corporate domain
  containers:
  - name: app
    image: myapp
```

---

## 🔊 Headless Services DNS

### Headless Service DNS Behavior

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         HEADLESS SERVICE DNS                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Regular Service (ClusterIP):                                                       │
│  nslookup nginx-service → 10.96.0.100 (single ClusterIP)                           │
│                                                                                      │
│  Headless Service (clusterIP: None):                                               │
│  nslookup mysql-headless → 10.244.1.5, 10.244.2.8, 10.244.3.2 (all pod IPs)       │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  Individual pod access (StatefulSet):                                              │
│                                                                                      │
│  mysql-0.mysql-headless.default.svc.cluster.local → 10.244.1.5                     │
│  mysql-1.mysql-headless.default.svc.cluster.local → 10.244.2.8                     │
│  mysql-2.mysql-headless.default.svc.cluster.local → 10.244.3.2                     │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Test headless service DNS
kubectl run test --image=busybox:1.28 --rm -it -- nslookup mysql-headless

# Output shows ALL pod IPs:
# Name:      mysql-headless
# Address 1: 10.244.1.5 mysql-0.mysql-headless.default.svc.cluster.local
# Address 2: 10.244.2.8 mysql-1.mysql-headless.default.svc.cluster.local
# Address 3: 10.244.3.2 mysql-2.mysql-headless.default.svc.cluster.local
```

---

## 🌍 External DNS

### Resolving External Names

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL DNS RESOLUTION                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. Pod queries google.com                                                          │
│  2. CoreDNS doesn't know it (not a service)                                        │
│  3. CoreDNS forwards to upstream DNS (usually node's DNS)                          │
│  4. Response returned to pod                                                        │
│                                                                                      │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐               │
│  │   Pod    │─────►│ CoreDNS  │─────►│  Node    │─────►│ External │               │
│  │          │      │          │      │   DNS    │      │   DNS    │               │
│  │ google.com?     │ forward  │      │ 8.8.8.8  │      │          │               │
│  └──────────┘      └──────────┘      └──────────┘      └──────────┘               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### ExternalName Service

```yaml
# Map internal name to external DNS
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: database.example.com
```

```bash
# DNS lookup returns CNAME
kubectl run test --image=busybox:1.28 --rm -it -- nslookup my-database
# Returns: database.example.com
```

---

## 📜 CoreDNS Configuration

### CoreDNS Corefile

```bash
# View CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```

```
# Default Corefile
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

### Corefile Plugins

| Plugin | Purpose |
|--------|---------|
| `errors` | Log errors |
| `health` | Health check endpoint |
| `ready` | Readiness probe endpoint |
| `kubernetes` | Kubernetes service discovery |
| `prometheus` | Metrics endpoint |
| `forward` | Forward unknown queries |
| `cache` | DNS response caching |
| `loop` | Detect and break loops |
| `reload` | Reload config on change |
| `loadbalance` | Round-robin DNS responses |

### Customize CoreDNS

```yaml
# Add custom DNS entries
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    example.com:53 {
        forward . 10.0.0.53
    }
```

---

## 🔧 Debugging DNS

### Debug Commands

```bash
# 1. Check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check CoreDNS service
kubectl get svc -n kube-system kube-dns

# 3. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. Test DNS from a pod
kubectl run dnstest --image=busybox:1.28 --rm -it -- nslookup kubernetes

# 5. Test with full FQDN
kubectl run dnstest --image=busybox:1.28 --rm -it -- nslookup kubernetes.default.svc.cluster.local

# 6. Check pod's resolv.conf
kubectl exec nginx-pod -- cat /etc/resolv.conf

# 7. DNS debugging pod
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
kubectl exec -it dnsutils -- nslookup kubernetes
kubectl exec -it dnsutils -- dig kubernetes.default.svc.cluster.local
```

### Debug Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
spec:
  containers:
  - name: dnsutils
    image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
    command:
    - sleep
    - "infinity"
```

---

## ❗ Common Issues

### DNS Troubleshooting Table

| Issue | Cause | Solution |
|-------|-------|----------|
| DNS not resolving | CoreDNS not running | Check CoreDNS pods |
| Timeout | Network Policy blocking | Allow port 53 to CoreDNS |
| NXDOMAIN | Wrong service name | Check service exists |
| Slow DNS | Missing ndots setting | Adjust ndots or use FQDN |
| External DNS fails | Forward not configured | Check Corefile forward |

### Common Fixes

```bash
# Issue: CoreDNS pods not running
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl describe pods -n kube-system -l k8s-app=kube-dns

# Issue: Service not found
kubectl get svc --all-namespaces | grep <service-name>

# Issue: Network Policy blocking DNS
# Ensure egress to port 53 is allowed
kubectl get networkpolicies -A

# Issue: Slow DNS (ndots)
# Use FQDN to skip search domain iteration
curl http://nginx.default.svc.cluster.local
```

---

## 🎓 CKA Exam Tips

### Quick DNS Testing

```bash
# Quick DNS test
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service-name>

# Test with wget
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- http://<service-name>

# Check resolv.conf
kubectl exec <pod> -- cat /etc/resolv.conf
```

### Key Points for Exam

1. **DNS format**: `<svc>.<ns>.svc.cluster.local`
2. **CoreDNS runs in**: `kube-system` namespace
3. **CoreDNS service**: `kube-dns` (ClusterIP 10.96.0.10)
4. **Headless service**: Returns pod IPs directly
5. **dnsPolicy default**: `ClusterFirst`
6. **Debug tool**: `busybox:1.28` with `nslookup`

### DNS Cheat Sheet

```bash
# Service DNS (same namespace)
nginx-service

# Service DNS (cross namespace)
nginx-service.production
nginx-service.production.svc.cluster.local

# Pod DNS (StatefulSet)
mysql-0.mysql-headless.default.svc.cluster.local

# Debug
nslookup <service>
dig <service>.default.svc.cluster.local
```

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **CoreDNS** | Cluster DNS server |
| **kube-dns** | CoreDNS service (10.96.0.10) |
| **Service DNS** | `<svc>.<ns>.svc.cluster.local` |
| **Pod DNS** | `<ip-dashed>.<ns>.pod.cluster.local` |
| **Headless DNS** | Returns all pod IPs |
| **dnsPolicy** | ClusterFirst, None, Default |

### DNS Resolution Order

1. Pod makes DNS query
2. Checks /etc/resolv.conf for nameserver
3. Query sent to CoreDNS (10.96.0.10)
4. CoreDNS checks if it's a Kubernetes service
5. Returns ClusterIP or forwards to upstream

### Essential Commands

```bash
# Test DNS
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service>

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# View resolv.conf
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

## 🎉 Phase 3: Networking Complete!

You've completed all networking chapters:

| Chapter | Topic | Status |
|---------|-------|--------|
| 12 | Services Deep Dive | ✅ |
| 13 | Ingress | ✅ |
| 14 | Network Policies | ✅ |
| 15 | DNS in Kubernetes | ✅ |

---

## 🔜 What's Next

In **Phase 4: Storage**, we'll cover:

- Chapter 16: Volumes
- Chapter 17: Persistent Volumes
- Chapter 18: ConfigMaps
- Chapter 19: Secrets

---

*DNS is the backbone of service discovery - understand it well!*

