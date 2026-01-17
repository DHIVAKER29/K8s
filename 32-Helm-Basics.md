# Chapter 32: Helm Basics

## Introduction

Installing a complex application in Kubernetes requires multiple YAML files: Deployments, Services, ConfigMaps, Secrets, Ingress, PVCs... **Helm** is the package manager for Kubernetes that bundles all these together into reusable **Charts**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE DEPLOYMENT PROBLEM                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Helm:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   To deploy WordPress:                                          │   │
│   │                                                                 │   │
│   │   kubectl apply -f wordpress-deployment.yaml                   │   │
│   │   kubectl apply -f wordpress-service.yaml                      │   │
│   │   kubectl apply -f wordpress-pvc.yaml                          │   │
│   │   kubectl apply -f wordpress-secret.yaml                       │   │
│   │   kubectl apply -f wordpress-configmap.yaml                    │   │
│   │   kubectl apply -f mysql-deployment.yaml                       │   │
│   │   kubectl apply -f mysql-service.yaml                          │   │
│   │   kubectl apply -f mysql-pvc.yaml                              │   │
│   │   kubectl apply -f mysql-secret.yaml                           │   │
│   │   kubectl apply -f ingress.yaml                                │   │
│   │                                                                 │   │
│   │   10+ files, manual configuration, hard to version/upgrade!    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Helm:                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   helm install my-wordpress bitnami/wordpress                  │   │
│   │                                                                 │   │
│   │   One command! Configurable! Upgradable! Rollback!             │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Helm Concepts

### 1.1 Key Terms

| Term | Definition |
|------|------------|
| **Chart** | Package of pre-configured Kubernetes resources |
| **Repository** | Collection of charts (like apt/yum repo) |
| **Release** | Instance of a chart running in cluster |
| **Values** | Configuration for customizing a chart |
| **Template** | Kubernetes manifests with Go templating |

### 1.2 Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HELM ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐                                                      │
│   │    User      │                                                      │
│   │  helm CLI    │                                                      │
│   └──────┬───────┘                                                      │
│          │                                                              │
│          │ 1. helm install                                              │
│          ▼                                                              │
│   ┌──────────────┐      ┌──────────────┐                               │
│   │    Chart     │ ───▶ │   Values     │                               │
│   │  (templates) │      │  (config)    │                               │
│   └──────┬───────┘      └──────┬───────┘                               │
│          │                     │                                        │
│          └─────────┬───────────┘                                        │
│                    │ 2. Render templates                                │
│                    ▼                                                    │
│          ┌──────────────────┐                                          │
│          │  Kubernetes YAML │                                          │
│          │    (manifests)   │                                          │
│          └────────┬─────────┘                                          │
│                   │ 3. Apply to cluster                                 │
│                   ▼                                                     │
│          ┌──────────────────┐                                          │
│          │   Kubernetes     │                                          │
│          │    Cluster       │                                          │
│          └──────────────────┘                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Installing Helm

### 2.1 Install Helm CLI

```bash
# macOS
brew install helm

# Linux (script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Linux (apt)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm

# Verify
helm version
```

### 2.2 Add Chart Repositories

```bash
# Add popular repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update repo cache
helm repo update

# List repositories
helm repo list

# Search for charts
helm search repo nginx
helm search repo mysql
```

---

## 3. Basic Helm Commands

### 3.1 Installing Charts

```bash
# Install a chart (creates a release)
helm install <release-name> <chart>

# Examples:
helm install my-nginx bitnami/nginx
helm install my-mysql bitnami/mysql
helm install my-wordpress bitnami/wordpress

# Install in specific namespace
helm install my-nginx bitnami/nginx -n web --create-namespace

# Install with custom values
helm install my-nginx bitnami/nginx --set replicaCount=3

# Install with values file
helm install my-nginx bitnami/nginx -f custom-values.yaml

# Install specific version
helm install my-nginx bitnami/nginx --version 13.2.0

# Dry run (see what would be created)
helm install my-nginx bitnami/nginx --dry-run
```

### 3.2 Managing Releases

```bash
# List releases
helm list
helm list -A                    # All namespaces
helm list -n <namespace>

# Get release status
helm status <release-name>

# Get release values
helm get values <release-name>
helm get values <release-name> --all    # Including defaults

# Get release manifest
helm get manifest <release-name>

# Get release history
helm history <release-name>
```

### 3.3 Upgrading Releases

```bash
# Upgrade a release
helm upgrade <release-name> <chart>

# Upgrade with new values
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Upgrade with values file
helm upgrade my-nginx bitnami/nginx -f new-values.yaml

# Install or upgrade (idempotent)
helm upgrade --install my-nginx bitnami/nginx
```

### 3.4 Rolling Back

```bash
# View history
helm history my-nginx

# Rollback to previous revision
helm rollback my-nginx

# Rollback to specific revision
helm rollback my-nginx 2
```

### 3.5 Uninstalling Releases

```bash
# Uninstall a release
helm uninstall <release-name>

# Keep history (allows rollback)
helm uninstall <release-name> --keep-history
```

---

## 4. Working with Values

### 4.1 View Default Values

```bash
# Show all configurable values
helm show values bitnami/nginx

# Save values to file
helm show values bitnami/nginx > default-values.yaml
```

### 4.2 Customizing Values

```bash
# Method 1: --set flag
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer

# Method 2: values file
cat <<EOF > my-values.yaml
replicaCount: 3
service:
  type: LoadBalancer
  port: 80
resources:
  limits:
    cpu: 100m
    memory: 128Mi
EOF

helm install my-nginx bitnami/nginx -f my-values.yaml

# Method 3: Multiple values files (later overrides earlier)
helm install my-nginx bitnami/nginx \
  -f base-values.yaml \
  -f prod-values.yaml
```

### 4.3 Common Values

| Value | Description | Example |
|-------|-------------|---------|
| `replicaCount` | Number of pods | `3` |
| `image.repository` | Container image | `nginx` |
| `image.tag` | Image version | `1.21.0` |
| `service.type` | Service type | `ClusterIP`, `NodePort`, `LoadBalancer` |
| `service.port` | Service port | `80` |
| `ingress.enabled` | Enable Ingress | `true` |
| `resources.limits` | Resource limits | `cpu: 100m, memory: 128Mi` |
| `persistence.enabled` | Enable PVC | `true` |

---

## 5. Chart Structure

### 5.1 Chart Directory Layout

```
mychart/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl    # Template helpers
│   ├── NOTES.txt       # Post-install notes
│   └── tests/          # Test pods
└── README.md           # Documentation
```

### 5.2 Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 0.1.0           # Chart version
appVersion: "1.0.0"      # Application version
dependencies:
  - name: mysql
    version: 9.0.0
    repository: https://charts.bitnami.com/bitnami
```

### 5.3 Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

---

## 6. Creating Your Own Chart

### 6.1 Create a Chart

```bash
# Create new chart
helm create mychart

# This creates:
# mychart/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# ├── templates/
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ...
# └── ...

# Edit templates and values as needed

# Lint the chart
helm lint mychart

# Test template rendering
helm template mychart

# Package the chart
helm package mychart
# Creates: mychart-0.1.0.tgz

# Install from local chart
helm install my-release ./mychart
```

### 6.2 Template Functions

```yaml
# Common template functions

# Release info
{{ .Release.Name }}          # Release name
{{ .Release.Namespace }}     # Namespace

# Chart info
{{ .Chart.Name }}            # Chart name
{{ .Chart.Version }}         # Chart version

# Values
{{ .Values.replicaCount }}   # From values.yaml

# Built-in functions
{{ .Values.name | upper }}              # UPPERCASE
{{ .Values.name | lower }}              # lowercase
{{ .Values.name | quote }}              # "quoted"
{{ .Values.list | toYaml | indent 2 }}  # YAML formatting
{{ default "nginx" .Values.image }}     # Default value

# Conditionals
{{- if .Values.ingress.enabled }}
# ingress config here
{{- end }}

# Loops
{{- range .Values.hosts }}
- {{ . }}
{{- end }}
```

---

## 7. Common Helm Charts

### 7.1 Popular Charts

| Chart | Description | Install Command |
|-------|-------------|-----------------|
| nginx-ingress | Ingress controller | `helm install nginx ingress-nginx/ingress-nginx` |
| prometheus | Monitoring | `helm install prom prometheus-community/prometheus` |
| grafana | Dashboards | `helm install grafana grafana/grafana` |
| mysql | Database | `helm install mysql bitnami/mysql` |
| postgresql | Database | `helm install pg bitnami/postgresql` |
| redis | Cache | `helm install redis bitnami/redis` |
| mongodb | NoSQL DB | `helm install mongo bitnami/mongodb` |
| elasticsearch | Search | `helm install es elastic/elasticsearch` |
| cert-manager | TLS certs | `helm install cert-manager jetstack/cert-manager` |

### 7.2 Example: Installing Prometheus Stack

```bash
# Add repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install with custom values
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

---

## 8. Command Reference

```bash
# REPOSITORIES
helm repo add <name> <url>      # Add repo
helm repo update                # Update cache
helm repo list                  # List repos
helm repo remove <name>         # Remove repo
helm search repo <keyword>      # Search charts

# INSTALLATION
helm install <release> <chart>  # Install chart
helm install <release> <chart> --set key=value
helm install <release> <chart> -f values.yaml
helm install <release> <chart> -n <namespace> --create-namespace
helm install <release> <chart> --dry-run

# MANAGEMENT
helm list                       # List releases
helm list -A                    # All namespaces
helm status <release>           # Release status
helm get values <release>       # Get values
helm get manifest <release>     # Get manifests
helm history <release>          # Release history

# UPGRADE & ROLLBACK
helm upgrade <release> <chart>  # Upgrade
helm upgrade --install <release> <chart>  # Install or upgrade
helm rollback <release> <revision>        # Rollback

# UNINSTALL
helm uninstall <release>        # Remove release

# CHART DEVELOPMENT
helm create <name>              # Create chart
helm lint <chart>               # Validate chart
helm template <chart>           # Render templates
helm package <chart>            # Package chart
helm show values <chart>        # Show default values
```

---

## 9. Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HELM BEST PRACTICES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   • Use --dry-run before installing                                    │
│   • Keep values files in version control                               │
│   • Use helm upgrade --install for idempotency                         │
│   • Pin chart versions in production                                   │
│   • Use namespaces to organize releases                                │
│   • Review helm show values before installing                          │
│                                                                         │
│   ❌ DON'T:                                                             │
│   • Use latest chart versions in production                            │
│   • Store secrets in values files (use sealed-secrets)                 │
│   • Forget to helm repo update                                         │
│   • Ignore helm lint warnings                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HELM SUMMARY                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   WHAT: Package manager for Kubernetes                                  │
│                                                                         │
│   WHY:                                                                  │
│   • Bundle multiple K8s resources into one package                     │
│   • Configurable deployments with values                               │
│   • Version control and rollback                                       │
│   • Reusable charts                                                    │
│                                                                         │
│   KEY COMMANDS:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  helm repo add <name> <url>    # Add chart repository           │   │
│   │  helm search repo <name>       # Find charts                    │   │
│   │  helm install <rel> <chart>    # Install chart                  │   │
│   │  helm upgrade <rel> <chart>    # Upgrade release                │   │
│   │  helm rollback <rel> <rev>     # Rollback                       │   │
│   │  helm uninstall <rel>          # Remove release                 │   │
│   │  helm list                     # List releases                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   KEY CONCEPTS:                                                         │
│   • Chart = Package of K8s manifests                                   │
│   • Release = Running instance of a chart                              │
│   • Values = Configuration for customization                           │
│   • Repository = Collection of charts                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**Chapter 32 Complete!** ✅

---

## 🎉 Tutorial Now TRULY Complete!

You now have comprehensive coverage of:
- **Docker**: 16 chapters
- **Kubernetes**: 32 chapters (including CKA essentials + real-world tools)

**Your learning journey is complete! Go ace that CKA exam!** 🏆

