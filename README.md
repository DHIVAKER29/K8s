# ☸️ Kubernetes Mastery - From Zero to CKA

> A comprehensive, in-depth guide to Kubernetes - from fundamentals to CKA (Certified Kubernetes Administrator) certification.

---

## 🎯 What Makes This Tutorial Different

This is not a surface-level tutorial. Each chapter provides:

| Feature | Description |
|---------|-------------|
| **📖 Definitions** | Clear, precise definitions for every concept |
| **📊 Diagrams** | ASCII diagrams to visualize architecture and flows |
| **💻 Commands** | Complete command references with examples |
| **🌍 Real-World** | Production scenarios and use cases |
| **🔗 Docker Mapping** | How Docker concepts translate to Kubernetes |
| **🎓 CKA Tips** | Exam-specific tips and common questions |
| **🛠️ Hands-On** | Practical exercises you can run |

---

## 📚 Complete Curriculum

### Phase 1: Kubernetes Fundamentals

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 01 | [Why Kubernetes?](./01-Why-Kubernetes.md) | ✅ Complete | The problems K8s solves, when to use it |
| 02 | [Kubernetes Architecture](./02-Kubernetes-Architecture.md) | ✅ Complete | Control plane, nodes, all components |
| 03 | [Installing Kubernetes](./03-Installing-Kubernetes.md) | ✅ Complete | Minikube, kind, kubeadm, cloud options |
| 04 | [kubectl Mastery](./04-kubectl-Mastery.md) | ✅ Complete | The essential CLI tool, all commands |
| 05 | [Kubernetes Objects & YAML](./05-Objects-and-YAML.md) | ✅ Complete | Understanding K8s resource definitions |

### Phase 2: Core Workloads

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 06 | [Pods Deep Dive](./06-Pods-Deep-Dive.md) | ✅ Complete | The smallest deployable unit |
| 07 | [ReplicaSets](./07-ReplicaSets.md) | ✅ Complete | Maintaining pod replicas |
| 08 | [Deployments](./08-Deployments.md) | ✅ Complete | Declarative updates, rollouts, rollbacks |
| 09 | [DaemonSets](./09-DaemonSets.md) | ✅ Complete | Running pods on every node |
| 10 | [StatefulSets](./10-StatefulSets.md) | ✅ Complete | Stateful applications |
| 11 | [Jobs & CronJobs](./11-Jobs-and-CronJobs.md) | ✅ Complete | Batch processing |

### Phase 3: Networking

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 12 | [Services Deep Dive](./12-Services-Deep-Dive.md) | ✅ Complete | ClusterIP, NodePort, LoadBalancer |
| 13 | [Ingress](./13-Ingress.md) | ✅ Complete | HTTP routing, TLS termination |
| 14 | [Network Policies](./14-Network-Policies.md) | ✅ Complete | Pod-level firewall rules |
| 15 | [DNS in Kubernetes](./15-DNS-in-Kubernetes.md) | ✅ Complete | Service discovery, CoreDNS |

### Phase 4: Storage

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 16 | [Volumes](./16-Volumes.md) | ✅ Complete | EmptyDir, HostPath, PV, PVC, StorageClasses |
| 17 | [ConfigMaps & Secrets](./17-ConfigMaps-and-Secrets.md) | ✅ Complete | Configuration and sensitive data management |

### Phase 5: Security

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 18 | [RBAC](./18-RBAC.md) | ✅ Complete | Role-Based Access Control, Service Accounts |
| 19 | [Security Contexts](./19-Security-Contexts.md) | ✅ Complete | Pod and container security settings |
| 20 | [Pod Security Standards](./20-Pod-Security-Standards.md) | ✅ Complete | Enforcing security policies |

### Phase 6: Advanced Topics

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 21 | [Resource Management](./21-Resource-Management.md) | ✅ Complete | Requests, limits, QoS |
| 22 | [Scheduling](./22-Scheduling.md) | ✅ Complete | Node selection, affinity, taints |
| 23 | [Probes](./23-Probes.md) | ✅ Complete | Liveness, readiness, startup |
| 24 | [Horizontal Pod Autoscaler](./24-HPA.md) | ✅ Complete | Automatic scaling |
| 25 | [Custom Resources (CRDs)](./25-CRDs.md) | ✅ Complete | Extending Kubernetes |

### Phase 7: Operations & Troubleshooting

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 26 | [Logging & Monitoring](./26-Logging-Monitoring.md) | ✅ Complete | Observability in K8s |
| 27 | [Troubleshooting](./27-Troubleshooting.md) | ✅ Complete | Debugging pods, nodes, clusters |
| 28 | [Cluster Maintenance](./28-Cluster-Maintenance.md) | ✅ Complete | Upgrades, backups, etcd |
| 29 | [CKA Exam Preparation](./29-CKA-Exam-Preparation.md) | ✅ Complete | Tips, practice, resources |

### Phase 8: Bonus Topics (CKA + Real-World)

| # | Topic | Status | Description |
|---|-------|--------|-------------|
| 30 | [Pod Disruption Budgets](./30-Pod-Disruption-Budgets.md) | ✅ Complete | HA during maintenance (CKA!) |
| 31 | [Static Pods](./31-Static-Pods.md) | ✅ Complete | Control plane components (CKA!) |
| 32 | [Helm Basics](./32-Helm-Basics.md) | ✅ Complete | Package manager (real-world essential) |
| 33 | [Helmfile](./33-Helmfile.md) | ✅ Complete | Declarative multi-release management |

---

## 🔧 Prerequisites

Before starting:

- ✅ **Docker fundamentals** (completed in DockerLearning folder)
- ✅ **Command line basics**
- ✅ **YAML syntax understanding**
- ⬜ **A Kubernetes cluster** (we'll set this up in Chapter 03)

---

## 🛠️ Lab Environment Options

| Option | Difficulty | Best For |
|--------|------------|----------|
| **Minikube** | Easy | Learning, single-node |
| **kind** | Easy | Local multi-node, CI |
| **Docker Desktop** | Easy | macOS/Windows with K8s |
| **k3s** | Easy | Lightweight, edge, IoT |
| **kubeadm** | Medium | Production-like setup |
| **EKS/GKE/AKS** | Medium | Cloud-managed clusters |

---

## 📖 How to Use This Guide

### For Learning

```
1. Read each chapter sequentially
2. Run all commands in your lab environment
3. Complete the exercises at the end of each chapter
4. Review the CKA tips sections
```

### For Reference

```
1. Use the search (Ctrl+F) to find specific topics
2. Check the command cheatsheets
3. Refer to the diagrams for architecture understanding
```

### For CKA Exam

```
1. Focus on chapters marked with 🎓 CKA Tips
2. Practice the hands-on exercises repeatedly
3. Time yourself - CKA is performance-based
4. Review the CKA Exam Preparation chapter
```

---

## 🏆 CKA Exam Domains

| Domain | Weight | Chapters |
|--------|--------|----------|
| Cluster Architecture | 25% | 02, 03, 32 |
| Workloads & Scheduling | 15% | 06-11, 26 |
| Services & Networking | 20% | 12-15 |
| Storage | 10% | 16-19 |
| Troubleshooting | 30% | 27, 30, 31 |

---

## 🔗 Docker to Kubernetes Mapping

Quick reference for Docker users:

| Docker Concept | Kubernetes Equivalent |
|----------------|----------------------|
| Container | Container (in a Pod) |
| docker run | Pod / Deployment |
| docker-compose | Deployment + Service |
| Docker network | Service + NetworkPolicy |
| Docker volume | PersistentVolume + PVC |
| docker logs | kubectl logs |
| docker exec | kubectl exec |
| Dockerfile | Container image (same!) |
| Docker Swarm | Kubernetes |

---

## 📅 Estimated Learning Time

| Phase | Chapters | Time |
|-------|----------|------|
| Fundamentals | 01-05 | 8-10 hours |
| Core Workloads | 06-11 | 10-12 hours |
| Networking | 12-15 | 6-8 hours |
| Storage | 16-17 | 4-6 hours |
| Security | 18-20 | 4-6 hours |
| Advanced | 21-25 | 8-10 hours |
| Operations | 26-29 | 6-8 hours |
| Bonus Topics | 30-33 | 5-7 hours |
| **Total** | **33 chapters** | **~55-65 hours** |

---

*Let's begin your Kubernetes journey! 🚀*

*Last Updated: January 2026*

