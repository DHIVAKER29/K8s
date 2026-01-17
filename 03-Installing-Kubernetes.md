# ☸️ Chapter 03: Installing Kubernetes

> A comprehensive guide to setting up Kubernetes clusters - from local development to production.

---

## 📚 Table of Contents

1. [Installation Options Overview](#-installation-options-overview)
2. [Prerequisites](#-prerequisites)
3. [Minikube (Recommended for Learning)](#-minikube-recommended-for-learning)
4. [kind (Kubernetes in Docker)](#-kind-kubernetes-in-docker)
5. [k3s (Lightweight Kubernetes)](#-k3s-lightweight-kubernetes)
6. [Docker Desktop Kubernetes](#-docker-desktop-kubernetes)
7. [kubeadm (Production-Style)](#-kubeadm-production-style)
8. [Cloud Managed Kubernetes](#-cloud-managed-kubernetes)
9. [kubectl Configuration](#-kubectl-configuration)
10. [Verifying Your Cluster](#-verifying-your-cluster)
11. [Troubleshooting Installation](#-troubleshooting-installation)
12. [CKA Exam Relevance](#-cka-exam-relevance)
13. [Summary](#-summary)

---

## 🔍 Installation Options Overview

### Comparison Table

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES INSTALLATION OPTIONS                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Option         │ Use Case            │ Difficulty │ Multi-Node │ Production        │
│  ───────────────┼─────────────────────┼────────────┼────────────┼───────────────────│
│  Minikube       │ Learning, Dev       │ Easy       │ Yes*       │ No                │
│  kind           │ CI/CD, Testing      │ Easy       │ Yes        │ No                │
│  k3s            │ Edge, IoT, Dev      │ Easy       │ Yes        │ Limited           │
│  Docker Desktop │ macOS/Windows Dev   │ Very Easy  │ No         │ No                │
│  kubeadm        │ On-prem, Learning   │ Medium     │ Yes        │ Yes               │
│  EKS/GKE/AKS    │ Production Cloud    │ Easy       │ Yes        │ Yes (Managed)     │
│  kOps           │ AWS Production      │ Medium     │ Yes        │ Yes               │
│  Rancher        │ Multi-cluster       │ Medium     │ Yes        │ Yes               │
│                                                                                      │
│  * Minikube supports multi-node but it's experimental                              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    WHICH INSTALLATION SHOULD I USE?                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Are you learning Kubernetes?                                                        │
│  │                                                                                   │
│  ├─ Yes → Do you have Docker already?                                               │
│  │        │                                                                          │
│  │        ├─ Yes → kind (fastest setup)                                             │
│  │        │                                                                          │
│  │        └─ No → Minikube (most features, best docs)                               │
│  │                                                                                   │
│  └─ No → Is this for production?                                                    │
│          │                                                                           │
│          ├─ Yes → Cloud? → EKS / GKE / AKS                                          │
│          │        On-prem? → kubeadm / Rancher                                      │
│          │                                                                           │
│          └─ No → Testing/CI → kind                                                  │
│                  Edge/IoT → k3s                                                     │
│                                                                                      │
│  For CKA Exam Practice:                                                              │
│  • kubeadm → Understanding cluster setup                                            │
│  • Minikube → Quick practice                                                        │
│  • Killer.sh (exam simulator) → Best preparation                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Prerequisites

### For All Installations

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           PREREQUISITES                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Hardware (Minimum):                                                                │
│  ├── CPU: 2 cores                                                                   │
│  ├── Memory: 2 GB (4 GB recommended)                                               │
│  └── Disk: 20 GB free space                                                         │
│                                                                                      │
│  Software:                                                                          │
│  ├── Container Runtime (Docker, Podman, or containerd)                             │
│  ├── Virtualization enabled (for Minikube with VM driver)                          │
│  └── kubectl (Kubernetes CLI)                                                       │
│                                                                                      │
│  Network:                                                                           │
│  ├── Internet access (for pulling images)                                          │
│  └── Ports available (6443, 10250, etc.)                                           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Install kubectl (Required for All)

> **Definition**: kubectl is the command-line tool for interacting with Kubernetes clusters. You'll use it for all cluster operations.

#### macOS

```bash
# Using Homebrew (recommended)
brew install kubectl

# Or download directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# For Apple Silicon (M1/M2/M3)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Linux

```bash
# Using package manager (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl

# Or download directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Windows

```powershell
# Using Chocolatey
choco install kubernetes-cli

# Or using winget
winget install -e --id Kubernetes.kubectl

# Or download directly
curl.exe -LO "https://dl.k8s.io/release/v1.29.0/bin/windows/amd64/kubectl.exe"
# Add to PATH
```

#### Verify Installation

```bash
kubectl version --client

# Output:
# Client Version: v1.29.0
# Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

---

## 🚀 Minikube (Recommended for Learning)

> **Definition**: Minikube is a tool that runs a single-node Kubernetes cluster locally. It's the most popular choice for learning and development.

### Why Minikube?

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           MINIKUBE FEATURES                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ✅ Full Kubernetes cluster (all features)                                          │
│  ✅ Multiple driver options (Docker, VirtualBox, HyperKit, etc.)                   │
│  ✅ Built-in add-ons (Dashboard, Ingress, Metrics Server)                          │
│  ✅ LoadBalancer support via `minikube tunnel`                                      │
│  ✅ Multi-node clusters (experimental)                                              │
│  ✅ Easy pause/resume                                                               │
│  ✅ Excellent documentation                                                         │
│  ✅ Most similar to real cluster                                                    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Installation

#### macOS

```bash
# Using Homebrew
brew install minikube

# Or download directly
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube

# For Apple Silicon
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

#### Linux

```bash
# Download and install
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Or using package manager
# Debian/Ubuntu
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

#### Windows

```powershell
# Using Chocolatey
choco install minikube

# Or using winget
winget install minikube

# Or download installer from:
# https://minikube.sigs.k8s.io/docs/start/
```

### Starting a Cluster

```bash
# Start with default settings (uses Docker driver if available)
minikube start

# Start with specific driver
minikube start --driver=docker
minikube start --driver=virtualbox
minikube start --driver=hyperkit  # macOS

# Start with specific resources
minikube start --cpus=4 --memory=8192

# Start with specific Kubernetes version
minikube start --kubernetes-version=v1.28.0

# Start with specific container runtime
minikube start --container-runtime=containerd

# Start a multi-node cluster (experimental)
minikube start --nodes=3

# Example: Full configuration
minikube start \
  --driver=docker \
  --cpus=4 \
  --memory=8192 \
  --kubernetes-version=v1.29.0 \
  --container-runtime=containerd \
  --addons=ingress,metrics-server,dashboard
```

### Minikube Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           MINIKUBE ARCHITECTURE                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Your Machine                                                                        │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │  ┌──────────────┐                                                            │ │
│  │  │   kubectl    │────────────────────────────────────┐                       │ │
│  │  │   Terminal   │                                    │                        │ │
│  │  └──────────────┘                                    │                        │ │
│  │                                                       │                        │ │
│  │  ┌───────────────────────────────────────────────────▼───────────────────┐   │ │
│  │  │                     MINIKUBE VM/Container                             │   │ │
│  │  │                                                                       │   │ │
│  │  │  ┌─────────────────────────────────────────────────────────────────┐ │   │ │
│  │  │  │                    KUBERNETES CLUSTER                           │ │   │ │
│  │  │  │                                                                 │ │   │ │
│  │  │  │  ┌───────────────────────────────────────────────────────────┐ │ │   │ │
│  │  │  │  │                   Control Plane                           │ │ │   │ │
│  │  │  │  │  API Server │ Scheduler │ Controller │ etcd              │ │ │   │ │
│  │  │  │  └───────────────────────────────────────────────────────────┘ │ │   │ │
│  │  │  │                                                                 │ │   │ │
│  │  │  │  ┌───────────────────────────────────────────────────────────┐ │ │   │ │
│  │  │  │  │                   Worker Components                       │ │ │   │ │
│  │  │  │  │  kubelet │ kube-proxy │ containerd                       │ │ │   │ │
│  │  │  │  └───────────────────────────────────────────────────────────┘ │ │   │ │
│  │  │  │                                                                 │ │   │ │
│  │  │  │  ┌───────┐ ┌───────┐ ┌───────┐                                │ │   │ │
│  │  │  │  │  Pod  │ │  Pod  │ │  Pod  │  (Your applications)          │ │   │ │
│  │  │  │  └───────┘ └───────┘ └───────┘                                │ │   │ │
│  │  │  │                                                                 │ │   │ │
│  │  │  └─────────────────────────────────────────────────────────────────┘ │   │ │
│  │  │                                                                       │   │ │
│  │  └───────────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Essential Minikube Commands

```bash
# Cluster Lifecycle
minikube start                  # Start cluster
minikube stop                   # Stop cluster (preserves state)
minikube pause                  # Pause cluster
minikube unpause                # Unpause cluster
minikube delete                 # Delete cluster completely
minikube delete --all           # Delete all clusters

# Cluster Information
minikube status                 # Check cluster status
minikube ip                     # Get cluster IP
minikube logs                   # View Minikube logs
minikube ssh                    # SSH into Minikube node

# Configuration
minikube config view            # View configuration
minikube config set cpus 4      # Set default CPUs
minikube config set memory 8192 # Set default memory

# Add-ons (Built-in extensions)
minikube addons list            # List available add-ons
minikube addons enable ingress  # Enable Ingress controller
minikube addons enable dashboard
minikube addons enable metrics-server
minikube addons disable ingress

# Dashboard
minikube dashboard              # Open K8s Dashboard in browser

# Services
minikube service list           # List services with URLs
minikube service my-service     # Open service in browser
minikube tunnel                 # Create tunnel for LoadBalancer

# Docker Environment
eval $(minikube docker-env)     # Use Minikube's Docker daemon
minikube docker-env             # Show Docker environment vars
```

### Minikube Add-ons

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           USEFUL MINIKUBE ADD-ONS                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Add-on           │ Purpose                          │ Enable Command               │
│  ─────────────────┼──────────────────────────────────┼──────────────────────────────│
│  dashboard        │ Web UI for cluster               │ minikube addons enable dash..│
│  ingress          │ Nginx Ingress Controller         │ minikube addons enable ingr..│
│  metrics-server   │ Resource metrics (for HPA)       │ minikube addons enable metr..│
│  storage-provisioner │ Dynamic PV provisioning       │ (enabled by default)         │
│  default-storageclass │ Default StorageClass         │ (enabled by default)         │
│  registry         │ In-cluster Docker registry       │ minikube addons enable regi..│
│  metallb          │ LoadBalancer for bare metal      │ minikube addons enable meta..│
│                                                                                      │
│  Enable multiple at start:                                                          │
│  minikube start --addons=ingress,metrics-server,dashboard                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🐳 kind (Kubernetes in Docker)

> **Definition**: kind (Kubernetes IN Docker) runs Kubernetes clusters using Docker containers as nodes. It's fast, lightweight, and perfect for CI/CD.

### Why kind?

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              KIND FEATURES                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ✅ Fastest setup (< 1 minute)                                                      │
│  ✅ Multi-node clusters easily                                                      │
│  ✅ Perfect for CI/CD pipelines                                                     │
│  ✅ Uses Docker containers (no VM needed)                                           │
│  ✅ Lightweight                                                                      │
│  ✅ Official Kubernetes SIG project                                                 │
│  ✅ Good for testing multi-node scenarios                                           │
│                                                                                      │
│  ❌ Less features than Minikube                                                     │
│  ❌ LoadBalancer requires extra setup                                               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# macOS (Homebrew)
brew install kind

# macOS/Linux (Go install)
go install sigs.k8s.io/kind@v0.20.0

# macOS/Linux (Binary)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-$(uname)-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Windows (Chocolatey)
choco install kind
```

### Creating Clusters

```bash
# Simple single-node cluster
kind create cluster

# Named cluster
kind create cluster --name my-cluster

# With specific Kubernetes version
kind create cluster --image kindest/node:v1.29.0

# Multi-node cluster (using config file)
kind create cluster --config kind-config.yaml
```

### kind Configuration File

```yaml
# kind-config.yaml
# Multi-node cluster with 1 control plane and 2 workers

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  # Control plane node
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  
  # Worker nodes
  - role: worker
  - role: worker

# Networking configuration
networking:
  # Disable default CNI to use custom one
  # disableDefaultCNI: true
  
  # Pod subnet
  podSubnet: "10.244.0.0/16"
  
  # Service subnet
  serviceSubnet: "10.96.0.0/12"
```

```bash
# Create cluster with config
kind create cluster --config kind-config.yaml --name multi-node
```

### kind Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                             KIND ARCHITECTURE                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Docker Host                                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │                                                                               │ │
│  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐   │ │
│  │  │ kind-control-plane  │  │  kind-worker        │  │  kind-worker2       │   │ │
│  │  │ (Docker Container)  │  │ (Docker Container)  │  │ (Docker Container)  │   │ │
│  │  │                     │  │                     │  │                     │   │ │
│  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │   │ │
│  │  │ │ Control Plane   │ │  │ │ kubelet         │ │  │ │ kubelet         │ │   │ │
│  │  │ │ Components      │ │  │ │ kube-proxy      │ │  │ │ kube-proxy      │ │   │ │
│  │  │ │ + kubelet       │ │  │ │ containerd      │ │  │ │ containerd      │ │   │ │
│  │  │ └─────────────────┘ │  │ └─────────────────┘ │  │ └─────────────────┘ │   │ │
│  │  │                     │  │                     │  │                     │   │ │
│  │  │ ┌───┐ ┌───┐ ┌───┐  │  │ ┌───┐ ┌───┐ ┌───┐  │  │ ┌───┐ ┌───┐ ┌───┐  │   │ │
│  │  │ │Pod│ │Pod│ │Pod│  │  │ │Pod│ │Pod│ │Pod│  │  │ │Pod│ │Pod│ │Pod│  │   │ │
│  │  │ └───┘ └───┘ └───┘  │  │ └───┘ └───┘ └───┘  │  │ └───┘ └───┘ └───┘  │   │ │
│  │  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘   │ │
│  │                                                                               │ │
│  │  Each "node" is a Docker container running Kubernetes components!            │ │
│  │                                                                               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Essential kind Commands

```bash
# Cluster Management
kind create cluster                    # Create default cluster
kind create cluster --name dev         # Create named cluster
kind delete cluster                    # Delete default cluster
kind delete cluster --name dev         # Delete named cluster
kind get clusters                      # List all clusters

# Cluster Information
kind get nodes                         # List nodes in cluster
kind get kubeconfig                    # Get kubeconfig

# Load Images into Cluster
kind load docker-image my-app:v1       # Load local image
kind load image-archive my-app.tar     # Load from tar

# Logs
kind export logs                       # Export logs to directory
kind export logs ./my-logs             # Export to specific dir
```

---

## 🪶 k3s (Lightweight Kubernetes)

> **Definition**: k3s is a lightweight, certified Kubernetes distribution designed for IoT, edge computing, and resource-constrained environments.

### Why k3s?

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              K3S FEATURES                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ✅ Single binary < 100MB                                                           │
│  ✅ Starts in seconds                                                               │
│  ✅ Low memory footprint (~512MB)                                                   │
│  ✅ Certified Kubernetes                                                            │
│  ✅ Built-in SQLite (no separate etcd)                                             │
│  ✅ Built-in load balancer                                                          │
│  ✅ ARM support (Raspberry Pi)                                                      │
│  ✅ Easy multi-node setup                                                           │
│                                                                                      │
│  Best for: Edge computing, IoT, Raspberry Pi, CI/CD                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# One-line install (Linux)
curl -sfL https://get.k3s.io | sh -

# With options
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

# Check status
sudo systemctl status k3s

# Get kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml

# Copy kubeconfig to user
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

### k3s Multi-Node Setup

```bash
# On master node - get token
sudo cat /var/lib/rancher/k3s/server/node-token

# On worker nodes - join cluster
curl -sfL https://get.k3s.io | K3S_URL=https://master-ip:6443 K3S_TOKEN=<token> sh -
```

### k3d (k3s in Docker)

```bash
# Install k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Or Homebrew
brew install k3d

# Create cluster
k3d cluster create my-cluster

# Multi-node
k3d cluster create my-cluster --servers 1 --agents 3

# With port mapping
k3d cluster create my-cluster -p "8080:80@loadbalancer"
```

---

## 🐋 Docker Desktop Kubernetes

> **Definition**: Docker Desktop includes a single-node Kubernetes cluster that can be enabled with one click. Simplest option for macOS/Windows users.

### Enable Kubernetes

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    DOCKER DESKTOP KUBERNETES SETUP                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. Open Docker Desktop                                                              │
│                                                                                      │
│  2. Go to Settings (⚙️ icon)                                                         │
│                                                                                      │
│  3. Select "Kubernetes" from sidebar                                                │
│                                                                                      │
│  4. Check "Enable Kubernetes"                                                        │
│                                                                                      │
│  5. Click "Apply & Restart"                                                          │
│                                                                                      │
│  6. Wait for Kubernetes to start (green indicator)                                  │
│                                                                                      │
│  7. Verify: kubectl get nodes                                                        │
│                                                                                      │
│  Note: Takes 2-5 minutes on first enable                                            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Limitations

```
✅ Pros:
  • Zero configuration
  • Integrated with Docker
  • Auto-updates

❌ Cons:
  • Single-node only
  • Limited configuration
  • Uses more resources
  • No add-ons like Minikube
```

---

## 🔧 kubeadm (Production-Style)

> **Definition**: kubeadm is a tool for bootstrapping a best-practices Kubernetes cluster. It's the official way to set up production clusters.

### When to Use kubeadm

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          KUBEADM USE CASES                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ✅ On-premises production clusters                                                 │
│  ✅ Learning cluster administration (CKA!)                                          │
│  ✅ Custom hardware/network requirements                                            │
│  ✅ Air-gapped environments                                                         │
│                                                                                      │
│  ❌ NOT for: Quick testing, CI/CD, learning K8s basics                             │
│                                                                                      │
│  CKA Exam: Understanding kubeadm is ESSENTIAL!                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Prerequisites (All Nodes)

```bash
# Disable swap (required!)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install container runtime (containerd)
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Install kubeadm, kubelet, kubectl

```bash
# Add Kubernetes repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install packages
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize Control Plane (Master Node)

```bash
# Initialize cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP>

# Example with more options
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=192.168.1.10 \
  --kubernetes-version=v1.29.0

# Save the output! It contains the join command for worker nodes
```

### Configure kubectl

```bash
# For regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
# Output: NAME         STATUS     ROLES           AGE   VERSION
#         master       NotReady   control-plane   1m    v1.29.0
```

### Install CNI (Network Plugin)

```bash
# Flannel (simple)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Or Calico (more features)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Or Weave
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# After CNI install, node should become Ready
kubectl get nodes
# Output: NAME         STATUS   ROLES           AGE   VERSION
#         master       Ready    control-plane   5m    v1.29.0
```

### Join Worker Nodes

```bash
# On worker nodes (command from kubeadm init output)
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>

# If you lost the token, generate a new one on master
kubeadm token create --print-join-command
```

### kubeadm Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          KUBEADM CLUSTER SETUP                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  STEP 1: Initialize Master                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  kubeadm init                                                               │   │
│  │  ├── Generates certificates (/etc/kubernetes/pki/)                         │   │
│  │  ├── Creates kubeconfig files                                               │   │
│  │  ├── Creates static pod manifests (/etc/kubernetes/manifests/)             │   │
│  │  │   ├── kube-apiserver.yaml                                               │   │
│  │  │   ├── kube-controller-manager.yaml                                      │   │
│  │  │   ├── kube-scheduler.yaml                                               │   │
│  │  │   └── etcd.yaml                                                         │   │
│  │  ├── Starts kubelet (which starts static pods)                             │   │
│  │  └── Generates join token                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  STEP 2: Install CNI                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  kubectl apply -f <CNI manifest>                                           │   │
│  │  └── Enables pod networking                                                 │   │
│  │  └── Node becomes "Ready"                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  STEP 3: Join Workers                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  kubeadm join                                                               │   │
│  │  ├── Validates token and CA cert                                           │   │
│  │  ├── Downloads cluster config from API server                              │   │
│  │  ├── Generates kubelet kubeconfig                                          │   │
│  │  └── Starts kubelet (registers with cluster)                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ☁️ Cloud Managed Kubernetes

### Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      MANAGED KUBERNETES COMPARISON                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Provider │ Service │ Control Plane │ Pricing                                       │
│  ─────────┼─────────┼───────────────┼─────────────────────────────────────────────  │
│  AWS      │ EKS     │ Managed       │ $0.10/hr + node costs                         │
│  GCP      │ GKE     │ Managed       │ Free tier (1 cluster), then $0.10/hr          │
│  Azure    │ AKS     │ Managed (Free)│ Pay only for nodes                            │
│  DigitalOcean │ DOKS│ Managed (Free)│ Pay only for nodes                            │
│  Linode   │ LKE     │ Managed (Free)│ Pay only for nodes                            │
│                                                                                      │
│  All provide:                                                                        │
│  • Managed control plane                                                            │
│  • Automatic upgrades                                                               │
│  • Integration with cloud services                                                  │
│  • High availability options                                                        │
│  • Managed node pools                                                               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### AWS EKS Quick Start

```bash
# Install eksctl
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Create cluster
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name my-nodes \
  --node-type t3.medium \
  --nodes 3

# Get credentials
aws eks update-kubeconfig --name my-cluster --region us-west-2

# Delete cluster
eksctl delete cluster --name my-cluster --region us-west-2
```

### GCP GKE Quick Start

```bash
# Install gcloud SDK and authenticate
gcloud auth login
gcloud config set project my-project

# Create cluster
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-medium

# Get credentials
gcloud container clusters get-credentials my-cluster --zone us-central1-a

# Delete cluster
gcloud container clusters delete my-cluster --zone us-central1-a
```

### Azure AKS Quick Start

```bash
# Login
az login

# Create resource group
az group create --name myResourceGroup --location eastus

# Create cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Delete cluster
az aks delete --resource-group myResourceGroup --name myAKSCluster
```

---

## ⚙️ kubectl Configuration

### kubeconfig File

> **Definition**: The kubeconfig file (~/.kube/config) stores cluster connection information, credentials, and context settings.

```yaml
# ~/.kube/config structure
apiVersion: v1
kind: Config
preferences: {}

# Clusters - define cluster endpoints
clusters:
  - name: minikube
    cluster:
      server: https://192.168.49.2:8443
      certificate-authority: /path/to/ca.crt
  
  - name: production
    cluster:
      server: https://prod-api.example.com:6443
      certificate-authority-data: LS0tLS1CRU...

# Users - define credentials
users:
  - name: minikube-user
    user:
      client-certificate: /path/to/client.crt
      client-key: /path/to/client.key
  
  - name: admin
    user:
      token: eyJhbGciOiJSUzI1NiIs...

# Contexts - combine cluster + user
contexts:
  - name: minikube
    context:
      cluster: minikube
      user: minikube-user
      namespace: default
  
  - name: production
    context:
      cluster: production
      user: admin
      namespace: prod

# Current context
current-context: minikube
```

### Managing Contexts

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context production

# Set default namespace for context
kubectl config set-context --current --namespace=my-namespace

# View full config
kubectl config view

# View specific cluster info
kubectl cluster-info
```

### Multiple Clusters

```bash
# Merge kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/config-prod:~/.kube/config-dev
kubectl config view --merge --flatten > ~/.kube/config.merged
mv ~/.kube/config.merged ~/.kube/config

# Or use --kubeconfig flag
kubectl --kubeconfig=/path/to/other/config get pods

# Use KUBECONFIG environment variable
export KUBECONFIG=/path/to/config
kubectl get pods
```

### Aliases and Shortcuts (Productivity)

```bash
# Add to ~/.bashrc or ~/.zshrc

# Short alias for kubectl
alias k='kubectl'

# Common commands
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'

# With namespace
alias kgpa='kubectl get pods --all-namespaces'

# Describe and logs
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias klf='kubectl logs -f'

# Apply and delete
alias ka='kubectl apply -f'
alias kd='kubectl delete -f'

# Context and namespace
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'

# Enable bash/zsh completion
source <(kubectl completion bash)  # bash
source <(kubectl completion zsh)   # zsh
```

---

## ✅ Verifying Your Cluster

### Essential Verification Commands

```bash
# 1. Check nodes
kubectl get nodes -o wide
# All nodes should be "Ready"

# 2. Check system pods
kubectl get pods -n kube-system
# All pods should be "Running"

# 3. Check component health
kubectl get componentstatuses  # Deprecated but useful

# 4. Cluster info
kubectl cluster-info

# 5. Test deployment
kubectl create deployment nginx --image=nginx
kubectl get pods
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get services

# 6. Test pod connectivity
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- nginx

# 7. Cleanup test
kubectl delete deployment nginx
kubectl delete service nginx
```

### Verification Checklist

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      CLUSTER VERIFICATION CHECKLIST                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  □ All nodes are Ready                                                              │
│    kubectl get nodes                                                                │
│                                                                                      │
│  □ All system pods are Running                                                       │
│    kubectl get pods -n kube-system                                                  │
│                                                                                      │
│  □ CoreDNS is working                                                               │
│    kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes           │
│                                                                                      │
│  □ Can create and access pods                                                        │
│    kubectl run nginx --image=nginx                                                  │
│    kubectl get pods                                                                 │
│                                                                                      │
│  □ Can create and access services                                                    │
│    kubectl expose pod nginx --port=80                                               │
│    kubectl get services                                                             │
│                                                                                      │
│  □ Pod-to-pod networking works                                                       │
│    kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- <pod-ip>            │
│                                                                                      │
│  □ kubectl context is correct                                                        │
│    kubectl config current-context                                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Troubleshooting Installation

### Common Issues

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      COMMON INSTALLATION ISSUES                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ISSUE: Node shows "NotReady"                                                        │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  Causes:                                                                             │
│  • CNI not installed                                                                │
│  • kubelet not running                                                              │
│  • Container runtime issues                                                         │
│                                                                                      │
│  Debug:                                                                              │
│  kubectl describe node <node-name>                                                  │
│  sudo systemctl status kubelet                                                      │
│  sudo journalctl -u kubelet -f                                                      │
│                                                                                      │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  ISSUE: Pods stuck in "Pending"                                                      │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  Causes:                                                                             │
│  • No schedulable nodes                                                             │
│  • Insufficient resources                                                           │
│  • Taints/tolerations mismatch                                                      │
│                                                                                      │
│  Debug:                                                                              │
│  kubectl describe pod <pod-name>                                                    │
│  kubectl get nodes                                                                  │
│  kubectl describe node <node-name> | grep -A5 "Conditions"                         │
│                                                                                      │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  ISSUE: Cannot connect to cluster                                                    │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  Causes:                                                                             │
│  • Wrong kubeconfig                                                                 │
│  • Cluster not running                                                              │
│  • Network/firewall issues                                                          │
│                                                                                      │
│  Debug:                                                                              │
│  kubectl config current-context                                                     │
│  kubectl cluster-info                                                               │
│  minikube status  # if using minikube                                              │
│                                                                                      │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  ISSUE: DNS not working                                                              │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  Causes:                                                                             │
│  • CoreDNS not running                                                              │
│  • CNI issues                                                                       │
│                                                                                      │
│  Debug:                                                                              │
│  kubectl get pods -n kube-system -l k8s-app=kube-dns                               │
│  kubectl logs -n kube-system -l k8s-app=kube-dns                                   │
│  kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🎓 CKA Exam Relevance

### What to Know for CKA

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      CKA INSTALLATION TOPICS                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  HIGH PRIORITY (expect questions):                                                   │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • kubeadm init and join commands                                                   │
│  • Cluster upgrades with kubeadm                                                    │
│  • Understanding static pod manifests (/etc/kubernetes/manifests/)                  │
│  • kubelet configuration (/var/lib/kubelet/config.yaml)                            │
│  • Certificate locations (/etc/kubernetes/pki/)                                     │
│                                                                                      │
│  MEDIUM PRIORITY:                                                                    │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • CNI installation                                                                 │
│  • kubeconfig file structure                                                        │
│  • Context management                                                               │
│                                                                                      │
│  CKA Tips:                                                                          │
│  ───────────────────────────────────────────────────────────────────────────────────│
│  • Bookmark kubernetes.io/docs for kubeadm commands                                 │
│  • Practice cluster upgrades                                                        │
│  • Know the systemctl commands for kubelet                                          │
│  • Understand certificate expiration and renewal                                    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Summary

### Quick Start Recommendations

| Scenario | Recommended | Command |
|----------|-------------|---------|
| **Learning K8s** | Minikube | `minikube start` |
| **Quick testing** | kind | `kind create cluster` |
| **CI/CD** | kind | `kind create cluster --config kind.yaml` |
| **CKA practice** | kubeadm | Multi-VM setup |
| **Production** | EKS/GKE/AKS | Cloud CLI tools |
| **Edge/IoT** | k3s | `curl -sfL https://get.k3s.io \| sh -` |

### Essential Commands Cheat Sheet

```bash
# Minikube
minikube start              # Start cluster
minikube stop               # Stop cluster
minikube dashboard          # Open dashboard
minikube addons enable X    # Enable addon

# kind
kind create cluster         # Create cluster
kind delete cluster         # Delete cluster
kind get clusters           # List clusters

# kubectl config
kubectl config get-contexts    # List contexts
kubectl config use-context X   # Switch context
kubectl config current-context # Show current

# Verification
kubectl get nodes              # Check nodes
kubectl get pods -A            # All pods
kubectl cluster-info           # Cluster info
```

---

## 🔜 What's Next

In **Chapter 04: kubectl Mastery**, we'll deep-dive into:

- All kubectl commands and subcommands
- Resource management with kubectl
- Output formatting and filtering
- Imperative vs declarative commands
- kubectl plugins and extensions
- Productivity tips and tricks

---

*Your cluster is ready! Let's learn how to use it effectively!*

