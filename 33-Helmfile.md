# Chapter 33: Helmfile

## Introduction

Managing many Helm releases manually is tedious. **Helmfile** is a declarative tool that lets you define all your Helm releases in a single YAML file and deploy them with one command: `helmfile sync`.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE MULTI-RELEASE PROBLEM                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without Helmfile:                                                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   # Deploy 10 services manually...                             │   │
│   │   helm install api ./charts/api -f values/api-dev.yaml         │   │
│   │   helm install db ./charts/mysql -f values/db-dev.yaml         │   │
│   │   helm install cache ./charts/redis -f values/redis-dev.yaml   │   │
│   │   helm install queue ./charts/kafka -f values/kafka-dev.yaml   │   │
│   │   ... 6 more commands ...                                      │   │
│   │                                                                 │   │
│   │   Tedious! Error-prone! Hard to version!                       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Helmfile:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   helmfile sync                                                 │   │
│   │                                                                 │   │
│   │   One command! Declarative! Git-versioned! Environment-aware!  │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. What is Helmfile?

### 1.1 Key Features

| Feature | Description |
|---------|-------------|
| **Declarative** | Define releases in YAML, not imperative commands |
| **Multi-Release** | Manage dozens of Helm releases together |
| **Environments** | Different configs for dev/staging/prod |
| **Templating** | Go templates in helmfile.yaml |
| **Diff** | Preview changes before applying |
| **Dependencies** | Control release order |
| **GitOps** | Version control your entire stack |

### 1.2 Helmfile vs Helm

| Aspect | Helm | Helmfile |
|--------|------|----------|
| **Scope** | Single release | Multiple releases |
| **Configuration** | Command-line flags | Declarative YAML |
| **Environments** | Manual -f switching | Built-in environments |
| **Dependencies** | Manual ordering | Automatic |
| **Diff before apply** | helm diff plugin | Built-in |

---

## 2. Installing Helmfile

```bash
# macOS
brew install helmfile

# Linux
wget https://github.com/helmfile/helmfile/releases/download/v0.158.0/helmfile_0.158.0_linux_amd64.tar.gz
tar xzf helmfile_0.158.0_linux_amd64.tar.gz
sudo mv helmfile /usr/local/bin/

# Verify installation
helmfile version

# Install required Helm plugins
helm plugin install https://github.com/databus23/helm-diff
```

---

## 3. Helmfile Structure

### 3.1 Basic helmfile.yaml

```yaml
# helmfile.yaml

# Helm repositories
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx

# Default settings for all releases
helmDefaults:
  wait: true
  timeout: 600
  createNamespace: true

# Helm releases
releases:
  - name: nginx
    namespace: web
    chart: bitnami/nginx
    version: 15.0.0
    values:
      - replicaCount: 3

  - name: redis
    namespace: cache
    chart: bitnami/redis
    version: 18.0.0
    values:
      - auth:
          enabled: false
```

### 3.2 Complete helmfile.yaml Structure

```yaml
# helmfile.yaml

# Helm defaults applied to all releases
helmDefaults:
  wait: true                    # Wait for resources to be ready
  timeout: 600                  # Timeout in seconds
  recreatePods: false           # Recreate pods on upgrade
  force: false                  # Force resource updates
  createNamespace: true         # Create namespace if not exists
  cleanupOnFail: true           # Delete new resources on failure
  historyMax: 3                 # Max release history

# Chart repositories
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: stable
    url: https://charts.helm.sh/stable

# Environment-specific values
environments:
  default:
    values:
      - env: dev
      - replicas: 1
  
  staging:
    values:
      - env: staging
      - replicas: 2
  
  production:
    values:
      - env: prod
      - replicas: 5

---
# Releases
releases:
  - name: api-{{ .Values.env }}
    namespace: api
    chart: ./charts/api
    values:
      - replicas: {{ .Values.replicas }}
      - environment: {{ .Values.env }}

  - name: web-{{ .Values.env }}
    namespace: web
    chart: ./charts/web
    values:
      - replicas: {{ .Values.replicas }}
```

---

## 4. Essential Commands

### 4.1 Core Commands

```bash
# Sync all releases (install/upgrade)
helmfile sync

# Show what would change (diff)
helmfile diff

# Apply changes (diff + sync)
helmfile apply

# List releases
helmfile list

# Delete all releases
helmfile destroy

# Template rendering (debug)
helmfile template
```

### 4.2 Command with Options

```bash
# Use specific environment
helmfile -e production sync
helmfile -e staging diff

# Use specific helmfile
helmfile -f helmfile.yaml sync
helmfile -f path/to/helmfile.yaml sync

# Target specific releases
helmfile -l name=api sync
helmfile -l namespace=web sync

# Dry run
helmfile sync --dry-run

# Concurrency (parallel releases)
helmfile sync --concurrency=5

# Skip certain phases
helmfile sync --skip-deps          # Skip dependency update
helmfile sync --skip-diff-on-install
```

### 4.3 Common Workflows

```bash
# Preview changes, then apply
helmfile diff
helmfile apply

# Deploy to specific environment
helmfile -e production apply

# Deploy only specific release
helmfile -l name=api-dev sync

# Destroy and recreate
helmfile destroy
helmfile sync
```

---

## 5. Environments

### 5.1 Environment Configuration

```yaml
# helmfile.yaml

environments:
  default:                      # Used when no -e specified
    values:
      - devstack_label: dev
      - ttl: 8h
      - replicas: 1
  
  staging:
    values:
      - devstack_label: staging
      - ttl: 24h
      - replicas: 2
  
  production:
    values:
      - devstack_label: prod
      - ttl: forever
      - replicas: 5
    secrets:
      - secrets/prod.yaml       # Encrypted secrets

---
releases:
  - name: api-{{ .Values.devstack_label }}
    namespace: api
    chart: ./charts/api
    values:
      - replicas: {{ .Values.replicas }}
      - ttl: {{ .Values.ttl }}
```

### 5.2 Using Environments

```bash
# Default environment
helmfile sync

# Staging environment
helmfile -e staging sync

# Production environment
helmfile -e production sync
```

### 5.3 Environment Values Files

```yaml
environments:
  production:
    values:
      - environments/production/values.yaml    # Load from file
      - db_host: prod-db.example.com           # Inline values
```

---

## 6. Templating

### 6.1 Go Templates in Helmfile

```yaml
releases:
  # Dynamic release name
  - name: api-{{ .Values.devstack_label }}
    namespace: api
    chart: ./charts/api
    
    # Conditional values
    values:
      - replicas: {{ .Values.replicas }}
      {{- if eq .Values.devstack_label "prod" }}
      - resources:
          limits:
            cpu: "2"
            memory: "4Gi"
      {{- else }}
      - resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
      {{- end }}
```

### 6.2 Template Functions

```yaml
releases:
  - name: api-{{ .Values.devstack_label }}
    values:
      # Random string
      - secret: {{ randAlphaNum 12 | lower }}
      
      # Default value
      - replicas: {{ .Values.replicas | default 1 }}
      
      # String manipulation
      - name: {{ .Values.name | upper }}
      
      # Conditionals
      {{- if .Values.enabled }}
      - feature: enabled
      {{- end }}
```

### 6.3 Accessing Environment Info

```yaml
releases:
  - name: api
    values:
      # Environment name
      - env: {{ .Environment.Name }}
      
      # Namespace from release
      - namespace: {{ .Namespace }}
      
      # Release name
      - release: {{ .Release.Name }}
```

---

## 7. Real-World Example

### 7.1 Your Company's Pattern

Based on your company's helmfile.yaml, here's the pattern explained:

```yaml
# helmfile.yaml

helmDefaults:
  cleanupOnFail: false
  wait: true
  recreatePods: true
  createNamespace: false
  timeout: 1200                 # 20 minutes
  historyMax: 1

environments:
  default:
    values:
      # Devstack label for ephemeral environments
      - devstack_label: myname
      # Time-to-live for resources
      - ttl: 8h
      # Random secret for databases
      - secret: {{ randAlphaNum 12 | lower }}

---
releases:
  # API Service
  - name: api-{{ .Values.devstack_label }}
    namespace: api
    chart: ./charts/api
    values:
      - image: <commit-hash>
      - devstack_label: {{ .Values.devstack_label }}
      - ttl: {{ .Values.ttl }}
      - ephemeral_db: true
      - secret: {{ .Values.secret }}

  # Database
  - name: mysql-{{ .Values.devstack_label }}
    namespace: api
    chart: ./charts/mysql
    values:
      - devstack_label: {{ .Values.devstack_label }}
      - ttl: {{ .Values.ttl }}
```

### 7.2 Deploying Your Stack

```bash
# Deploy your devstack
helmfile sync

# Check what's deployed
helmfile list

# See what would change
helmfile diff

# Tear down when done
helmfile destroy
```

---

## 8. Dependencies and Ordering

### 8.1 Release Dependencies

```yaml
releases:
  # Database first
  - name: mysql
    namespace: db
    chart: bitnami/mysql
    
  # API depends on database
  - name: api
    namespace: api
    chart: ./charts/api
    needs:
      - db/mysql              # namespace/release format
    
  # Frontend depends on API
  - name: frontend
    namespace: web
    chart: ./charts/frontend
    needs:
      - api/api
```

### 8.2 Parallel vs Sequential

```bash
# Sequential (default, respects needs)
helmfile sync

# Parallel with concurrency
helmfile sync --concurrency=5

# Serial (one at a time)
helmfile sync --concurrency=1
```

---

## 9. Selectors

### 9.1 Label Selectors

```yaml
releases:
  - name: api
    labels:
      app: api
      tier: backend
    chart: ./charts/api

  - name: web
    labels:
      app: web
      tier: frontend
    chart: ./charts/web
```

```bash
# Deploy only backend
helmfile -l tier=backend sync

# Deploy specific app
helmfile -l app=api sync

# Multiple labels (AND)
helmfile -l tier=backend -l app=api sync
```

### 9.2 Common Selector Patterns

```bash
# By release name
helmfile -l name=api sync

# By namespace
helmfile -l namespace=web sync

# By chart
helmfile -l chart=bitnami/nginx sync
```

---

## 10. Command Reference

```bash
# CORE COMMANDS
helmfile sync                   # Install/upgrade all releases
helmfile diff                   # Show changes without applying
helmfile apply                  # Diff + sync (interactive)
helmfile template               # Render templates (debug)
helmfile list                   # List releases
helmfile status                 # Show release statuses
helmfile destroy                # Delete all releases

# OPTIONS
-e, --environment <name>        # Use specific environment
-f, --file <path>               # Use specific helmfile
-l, --selector <label=value>    # Filter releases by label
--concurrency <n>               # Parallel releases
--dry-run                       # Don't actually apply

# EXAMPLES
helmfile -e production sync
helmfile -l name=api diff
helmfile -f custom.yaml apply
helmfile sync --concurrency=3
helmfile destroy -l tier=backend
```

---

## 11. Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HELMFILE BEST PRACTICES                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   • Use environments for dev/staging/prod                              │
│   • Use `helmfile diff` before `helmfile sync`                         │
│   • Use labels for selective deployments                               │
│   • Store helmfile.yaml in version control                             │
│   • Use `needs` for dependency ordering                                │
│   • Use templating for dynamic values                                  │
│   • Set appropriate timeouts                                           │
│                                                                         │
│   ❌ DON'T:                                                             │
│   • Store secrets in plain text (use helm-secrets)                     │
│   • Ignore diff output                                                 │
│   • Skip wait in production                                            │
│   • Use high concurrency without testing                               │
│   • Hardcode environment-specific values                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Troubleshooting

### 12.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "release not found" | New release | Normal, will be installed |
| Timeout | Slow pod startup | Increase `timeout` |
| Diff plugin missing | Not installed | `helm plugin install https://github.com/databus23/helm-diff` |
| Template error | Syntax issue | Run `helmfile template` to debug |

### 12.2 Debug Commands

```bash
# Debug template rendering
helmfile template

# Verbose output
helmfile --debug sync

# Check specific release
helmfile -l name=api template

# Validate helmfile.yaml
helmfile lint
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HELMFILE SUMMARY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   WHAT: Declarative tool for managing multiple Helm releases           │
│                                                                         │
│   WHY:                                                                  │
│   • Manage 10s or 100s of releases in one file                         │
│   • Environment-specific configurations                                │
│   • Preview changes with diff                                          │
│   • GitOps-friendly                                                    │
│                                                                         │
│   KEY COMMANDS:                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  helmfile sync              # Install/upgrade all               │   │
│   │  helmfile diff              # Preview changes                   │   │
│   │  helmfile apply             # Diff + sync                       │   │
│   │  helmfile -e prod sync      # Use production env                │   │
│   │  helmfile -l name=api sync  # Deploy specific release           │   │
│   │  helmfile destroy           # Remove all releases               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   YOUR WORKFLOW:                                                        │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  1. Edit helmfile.yaml (uncomment services you need)           │   │
│   │  2. helmfile diff           (see what will change)             │   │
│   │  3. helmfile sync           (deploy everything)                │   │
│   │  4. helmfile destroy        (cleanup when done)                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**Chapter 33 Complete!** ✅

Now you understand the `helmfile sync` command your company uses daily!

