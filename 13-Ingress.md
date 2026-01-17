# ☸️ Chapter 13: Ingress

> HTTP/HTTPS routing to services - the smart way to expose your applications to the internet.

---

## 📚 Table of Contents

1. [What is Ingress?](#-what-is-ingress)
2. [Why Ingress?](#-why-ingress)
3. [Ingress vs Service Types](#-ingress-vs-service-types)
4. [Ingress Architecture](#-ingress-architecture)
5. [Ingress Controllers](#-ingress-controllers)
6. [Creating Ingress](#-creating-ingress)
7. [Path-Based Routing](#-path-based-routing)
8. [Host-Based Routing](#-host-based-routing)
9. [TLS/HTTPS](#-tlshttps)
10. [Ingress Annotations](#-ingress-annotations)
11. [Default Backend](#-default-backend)
12. [Ingress Classes](#-ingress-classes)
13. [Common Configurations](#-common-configurations)
14. [Operations](#-operations)
15. [Troubleshooting](#-troubleshooting)
16. [CKA Exam Tips](#-cka-exam-tips)
17. [Summary](#-summary)

---

## 📖 What is Ingress?

### Definition

> **Ingress** is an API object that manages external access to services in a cluster, typically HTTP/HTTPS. It provides load balancing, SSL termination, and name-based virtual hosting.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS INGRESS?                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Ingress = HTTP/HTTPS Router + Load Balancer + SSL Terminator                       │
│                                                                                      │
│  Internet                                                                            │
│      │                                                                               │
│      │  https://myapp.example.com/api                                               │
│      │  https://myapp.example.com/web                                               │
│      │                                                                               │
│      ▼                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Ingress Controller                                    │ │
│  │                    (nginx, traefik, haproxy, etc.)                           │ │
│  │                                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │                     Ingress Rules                                       │ │ │
│  │  │                                                                         │ │ │
│  │  │  /api  → api-service:80                                                │ │ │
│  │  │  /web  → web-service:80                                                │ │ │
│  │  │  /     → default-service:80                                            │ │ │
│  │  │                                                                         │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                       │
│              ┌───────────────┼───────────────┐                                      │
│              ▼               ▼               ▼                                      │
│        ┌──────────┐   ┌──────────┐   ┌──────────┐                                  │
│        │api-svc   │   │web-svc   │   │default   │                                  │
│        │  :80     │   │  :80     │   │  :80     │                                  │
│        └──────────┘   └──────────┘   └──────────┘                                  │
│              │               │               │                                      │
│              ▼               ▼               ▼                                      │
│           [Pods]          [Pods]          [Pods]                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Ingress Capabilities

| Capability | Description |
|------------|-------------|
| **Path-based routing** | Route `/api` to API service, `/web` to web service |
| **Host-based routing** | Route `api.example.com` to API, `web.example.com` to web |
| **TLS termination** | Handle HTTPS, decrypt at ingress level |
| **Load balancing** | Distribute traffic across pods |
| **Name-based virtual hosting** | Multiple domains on one IP |

---

## ❓ Why Ingress?

### The Problem with LoadBalancer Services

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    WITHOUT INGRESS (LoadBalancer per service)                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Internet                                                                           │
│      │                                                                               │
│      ├─────────────────────┬─────────────────────┬─────────────────────┐            │
│      │                     │                     │                     │            │
│      ▼                     ▼                     ▼                     ▼            │
│  ┌─────────┐          ┌─────────┐          ┌─────────┐          ┌─────────┐        │
│  │   LB    │          │   LB    │          │   LB    │          │   LB    │        │
│  │ $20/mo  │          │ $20/mo  │          │ $20/mo  │          │ $20/mo  │        │
│  └────┬────┘          └────┬────┘          └────┬────┘          └────┬────┘        │
│       │                    │                    │                    │              │
│       ▼                    ▼                    ▼                    ▼              │
│  ┌─────────┐          ┌─────────┐          ┌─────────┐          ┌─────────┐        │
│  │API Svc  │          │Web Svc  │          │Auth Svc │          │Admin Svc│        │
│  └─────────┘          └─────────┘          └─────────┘          └─────────┘        │
│                                                                                      │
│  Problems:                                                                          │
│  • 4 Load Balancers = 4x cost ($80/month)                                          │
│  • 4 External IPs                                                                   │
│  • 4 DNS records to manage                                                         │
│  • No centralized SSL management                                                   │
│  • No unified routing                                                              │
│                                                                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                    WITH INGRESS (Single entry point)                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Internet                                                                           │
│      │                                                                               │
│      ▼                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                        Single Load Balancer                                 │   │
│  │                        (or NodePort)                                        │   │
│  │                        $20/mo                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                              │                                                       │
│                              ▼                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      Ingress Controller                                     │   │
│  │                                                                             │   │
│  │  /api   → API Service                                                      │   │
│  │  /web   → Web Service                                                      │   │
│  │  /auth  → Auth Service                                                     │   │
│  │  /admin → Admin Service                                                    │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Benefits:                                                                          │
│  • 1 Load Balancer = $20/month (75% savings!)                                      │
│  • 1 External IP                                                                    │
│  • 1 DNS record (with path/host routing)                                           │
│  • Centralized SSL termination                                                     │
│  • Unified routing configuration                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Ingress vs Service Types

### Comparison

| Aspect | NodePort | LoadBalancer | Ingress |
|--------|----------|--------------|---------|
| **Protocol** | TCP/UDP | TCP/UDP | HTTP/HTTPS |
| **Layer** | L4 | L4 | L7 |
| **Cost** | Free | Per LB | One LB for all |
| **Path routing** | ❌ | ❌ | ✅ |
| **Host routing** | ❌ | ❌ | ✅ |
| **SSL termination** | ❌ | Per LB | ✅ Centralized |
| **Use case** | Dev/test | Single service | Multiple services |

---

## 🏗️ Ingress Architecture

### Components

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         INGRESS ARCHITECTURE                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Two components needed:                                                             │
│                                                                                      │
│  1. INGRESS RESOURCE (YAML definition)                                             │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Defines routing rules                                                            │
│  • Created by you (the user)                                                       │
│  • Declarative: "I want /api to go to api-service"                                │
│                                                                                      │
│  2. INGRESS CONTROLLER (Running pod)                                               │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  • Reads Ingress resources                                                         │
│  • Implements the rules                                                            │
│  • Usually nginx, traefik, haproxy, or cloud-specific                             │
│  • Must be installed separately!                                                   │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                    Ingress Controller (Pod)                                 │   │
│  │                                                                             │   │
│  │   ┌──────────────────┐                                                      │   │
│  │   │ Watches API for  │                                                      │   │
│  │   │ Ingress resources│────────────────────────────────────────────┐        │   │
│  │   └──────────────────┘                                            │        │   │
│  │                                                                   ▼        │   │
│  │   ┌──────────────────┐     ┌──────────────────────────────────────────┐   │   │
│  │   │ nginx.conf       │◄────│  Ingress Resource: my-ingress            │   │   │
│  │   │ (generated)      │     │  rules:                                   │   │   │
│  │   │                  │     │  - path: /api → api-service              │   │   │
│  │   │ location /api {  │     │  - path: /web → web-service              │   │   │
│  │   │   proxy_pass ... │     └──────────────────────────────────────────┘   │   │
│  │   │ }                │                                                     │   │
│  │   └──────────────────┘                                                     │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🎮 Ingress Controllers

### Popular Controllers

| Controller | Provider | Description |
|------------|----------|-------------|
| **nginx-ingress** | Community/F5 | Most popular, feature-rich |
| **traefik** | Traefik Labs | Modern, auto-discovery |
| **haproxy** | HAProxy | High performance |
| **AWS ALB** | AWS | Native AWS integration |
| **GCE** | Google | Native GCP integration |
| **Azure AGIC** | Azure | Native Azure integration |
| **Contour** | VMware | Envoy-based |
| **Kong** | Kong Inc | API Gateway features |

### Installing NGINX Ingress Controller

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

# Using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Check if controller is ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## 🛠️ Creating Ingress

### Basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Imperative Creation

```bash
# Create ingress
kubectl create ingress simple --rule="/*=web-service:80"

# With host
kubectl create ingress myapp --rule="myapp.example.com/*=web-service:80"

# Generate YAML
kubectl create ingress myapp --rule="myapp.example.com/*=web-service:80" \
  --dry-run=client -o yaml > ingress.yaml
```

---

## 🛤️ Path-Based Routing

### Multiple Paths

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-service
            port:
              number: 80
```

### Path Types

| PathType | Behavior | Example |
|----------|----------|---------|
| `Prefix` | Matches path prefix | `/api` matches `/api`, `/api/v1`, `/api/users` |
| `Exact` | Matches exact path | `/api` only matches `/api`, not `/api/v1` |
| `ImplementationSpecific` | Controller decides | Depends on ingress controller |

### Path Routing Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         PATH-BASED ROUTING                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   Incoming Request                                                                  │
│        │                                                                             │
│        ▼                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────────────┐  │
│   │                      Ingress Controller                                     │  │
│   │                                                                             │  │
│   │  URL: example.com/api/users                                                │  │
│   │                     │                                                       │  │
│   │                     ▼                                                       │  │
│   │  ┌─────────────────────────────────────────────────────────────────────┐   │  │
│   │  │  Rule matching:                                                     │   │  │
│   │  │                                                                     │   │  │
│   │  │  /api/*   → api-service:80      ✅ MATCH!                          │   │  │
│   │  │  /web/*   → web-service:80                                         │   │  │
│   │  │  /        → default-service:80                                     │   │  │
│   │  │                                                                     │   │  │
│   │  └─────────────────────────────────────────────────────────────────────┘   │  │
│   │                                                                             │  │
│   └─────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                       │
│                              ▼                                                       │
│                        ┌──────────┐                                                 │
│                        │api-svc   │                                                 │
│                        │  :80     │                                                 │
│                        └──────────┘                                                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🌐 Host-Based Routing

### Multiple Hosts

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

### Host Routing Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         HOST-BASED ROUTING                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   All domains point to same IP (Ingress Controller)                                │
│                                                                                      │
│   api.example.com ─────┐                                                            │
│   web.example.com ─────┼───► 52.23.45.67 (Ingress Controller)                      │
│   admin.example.com ───┘                                                            │
│                              │                                                       │
│                              ▼                                                       │
│   ┌─────────────────────────────────────────────────────────────────────────────┐  │
│   │                      Ingress Controller                                     │  │
│   │                                                                             │  │
│   │  Host: api.example.com    → api-service:80                                 │  │
│   │  Host: web.example.com    → web-service:80                                 │  │
│   │  Host: admin.example.com  → admin-service:80                               │  │
│   │                                                                             │  │
│   └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│   DNS Configuration:                                                                │
│   api.example.com     A    52.23.45.67                                             │
│   web.example.com     A    52.23.45.67                                             │
│   admin.example.com   A    52.23.45.67                                             │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Combined Host and Path

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
```

---

## 🔐 TLS/HTTPS

### TLS Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret     # Contains certificate
  
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### Creating TLS Secret

```bash
# Generate self-signed certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.example.com"

# Create secret
kubectl create secret tls myapp-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Verify
kubectl get secret myapp-tls-secret
kubectl describe secret myapp-tls-secret
```

### TLS Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         TLS TERMINATION                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   Client                                                                            │
│     │                                                                                │
│     │  HTTPS (encrypted)                                                            │
│     │                                                                                │
│     ▼                                                                                │
│   ┌─────────────────────────────────────────────────────────────────────────────┐  │
│   │                      Ingress Controller                                     │  │
│   │                                                                             │  │
│   │  1. Receive HTTPS request                                                  │  │
│   │  2. Decrypt using TLS secret                                               │  │
│   │  3. Route based on rules                                                   │  │
│   │                                                                             │  │
│   └─────────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                                │
│     │  HTTP (unencrypted, internal)                                                 │
│     │                                                                                │
│     ▼                                                                                │
│   ┌──────────┐                                                                      │
│   │ Service  │                                                                      │
│   └──────────┘                                                                      │
│                                                                                      │
│   SSL/TLS is terminated at Ingress Controller                                      │
│   Internal traffic is HTTP (faster, simpler)                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Ingress Annotations

### Common NGINX Ingress Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Rewrite path
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # Body size
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"
    
    # Sticky sessions
    nginx.ingress.kubernetes.io/affinity: "cookie"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    
    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Custom-Header "value";
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### URL Rewrite Example

```yaml
# /api/users → backend receives /users
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 🏠 Default Backend

### Definition

> **Default Backend** handles requests that don't match any rule.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 🏷️ Ingress Classes

### What is IngressClass?

> **IngressClass** specifies which controller should handle an Ingress resource. Important when multiple controllers are installed.

```yaml
# IngressClass definition
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx

---

# Ingress using the class
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx     # Specify which controller
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

```bash
# List ingress classes
kubectl get ingressclass
```

---

## ⚙️ Common Configurations

### Complete Production Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  
  tls:
  - hosts:
    - api.myapp.com
    - www.myapp.com
    secretName: myapp-tls
  
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  
  - host: www.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 🔧 Operations

### Common Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl create ingress myingress --rule="myapp.com/*=myservice:80"
kubectl apply -f ingress.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get ingress
kubectl get ing                    # Short form
kubectl get ing -o wide

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe ingress myingress

# ═══════════════════════════════════════════════════════════════════
# VIEW INGRESS CLASSES
# ═══════════════════════════════════════════════════════════════════
kubectl get ingressclass

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete ingress myingress
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 404 Not Found | Path/host mismatch | Check rules match request |
| 502 Bad Gateway | Backend service down | Check service/pods |
| 503 Service Unavailable | No endpoints | Check selector matches pods |
| No ADDRESS | Controller issue | Check ingress controller |

### Debug Commands

```bash
# Check ingress
kubectl describe ingress myingress

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Check backend service
kubectl get svc myservice
kubectl get endpoints myservice

# Test from within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- http://myservice
```

---

## 🎓 CKA Exam Tips

### Quick Ingress Creation

```bash
# Create ingress with rule
kubectl create ingress myingress --rule="myapp.com/*=myservice:80"

# With TLS
kubectl create ingress myingress \
  --rule="myapp.com/*=myservice:80,tls=myapp-tls"

# Generate YAML
kubectl create ingress myingress --rule="myapp.com/*=myservice:80" \
  --dry-run=client -o yaml > ingress.yaml
```

### Key Points

1. **Ingress Controller required** - not installed by default
2. **pathType is required** (Prefix, Exact, ImplementationSpecific)
3. **TLS secret must be type kubernetes.io/tls**
4. **ingressClassName for multiple controllers**
5. **Service must exist for backend**

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Ingress** | L7 HTTP/HTTPS router |
| **Ingress Controller** | Implements routing rules |
| **Path-based routing** | Route by URL path |
| **Host-based routing** | Route by hostname |
| **TLS termination** | HTTPS at ingress |
| **IngressClass** | Select controller |

### Ingress vs LoadBalancer

| Aspect | LoadBalancer | Ingress |
|--------|--------------|---------|
| Layer | L4 | L7 |
| Cost | Per service | Shared |
| Routing | None | Path/Host |
| SSL | Per LB | Centralized |

---

## 🔜 What's Next

In **Chapter 14: Network Policies**, we'll cover:

- Pod-to-pod traffic control
- Ingress and egress rules
- Namespace isolation
- Default deny policies

---

*Ingress is essential for production HTTP services - master it!*

