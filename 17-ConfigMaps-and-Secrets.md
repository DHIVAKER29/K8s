# Chapter 17: ConfigMaps & Secrets - Configuration Management

## Introduction

Applications need configuration. Database URLs, feature flags, API keys, certificates - all of this data needs to be injected into your containers somehow. Kubernetes provides two primary objects for this: **ConfigMaps** for non-sensitive data and **Secrets** for sensitive data.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGURATION MANAGEMENT                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Traditional Approach (Anti-Pattern):                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      Docker Image                               │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  Application Code                                       │   │   │
│   │   │  +                                                      │   │   │
│   │   │  Configuration (hardcoded)                              │   │   │
│   │   │  +                                                      │   │   │
│   │   │  Secrets (EXPOSED!)                                     │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   Problems:                                                     │   │
│   │   ❌ Different image per environment                           │   │
│   │   ❌ Secrets in source control                                 │   │
│   │   ❌ No way to update without rebuild                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Kubernetes Approach (Best Practice):                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐    │   │
│   │   │  ConfigMap    │   │    Secret     │   │ Docker Image  │    │   │
│   │   │               │   │               │   │               │    │   │
│   │   │ DB_HOST=...   │   │ DB_PASS=...   │   │  Application  │    │   │
│   │   │ LOG_LEVEL=... │   │ API_KEY=...   │   │  Code Only    │    │   │
│   │   │ config.yaml   │   │ tls.crt       │   │               │    │   │
│   │   └───────┬───────┘   └───────┬───────┘   └───────┬───────┘    │   │
│   │           │                   │                   │            │   │
│   │           └───────────────────┼───────────────────┘            │   │
│   │                               ▼                                │   │
│   │                    ┌─────────────────────┐                     │   │
│   │                    │        POD          │                     │   │
│   │                    │                     │                     │   │
│   │                    │  Injected at        │                     │   │
│   │                    │  runtime via:       │                     │   │
│   │                    │  • Env vars         │                     │   │
│   │                    │  • Volume mounts    │                     │   │
│   │                    └─────────────────────┘                     │   │
│   │                                                                 │   │
│   │   Benefits:                                                     │   │
│   │   ✅ Same image, different config per environment              │   │
│   │   ✅ Secrets managed separately                                │   │
│   │   ✅ Update config without rebuilding                          │   │
│   │   ✅ Separation of concerns                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. ConfigMaps

### 1.1 What is a ConfigMap?

A **ConfigMap** is a Kubernetes API object that stores non-confidential configuration data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or configuration files.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CONFIGMAP ANATOMY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   apiVersion: v1                                                        │
│   kind: ConfigMap                                                       │
│   metadata:                                                             │
│     name: my-config                                                     │
│     namespace: default                                                  │
│   data:                        ◀── For UTF-8 text data                 │
│     DATABASE_HOST: "db.example.com"    ◀── Simple key-value           │
│     LOG_LEVEL: "info"                                                   │
│     config.json: |             ◀── Multi-line file content            │
│       {                                                                 │
│         "key": "value"                                                  │
│       }                                                                 │
│   binaryData:                  ◀── For binary data (base64 encoded)   │
│     logo.png: "iVBORw0KGgo..."                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 ConfigMap Properties

| Property | Description |
|----------|-------------|
| **Max Size** | 1 MiB (1,048,576 bytes) |
| **Scope** | Namespace-scoped |
| **Data Types** | `data` (string) or `binaryData` (base64) |
| **Immutable** | Optional - once set, cannot be changed |
| **Updates** | Changes reflected in pods (eventually) |

### 1.3 Creating ConfigMaps

#### Method 1: Imperative - From Literals

```bash
# Single key-value
kubectl create configmap my-config --from-literal=DATABASE_HOST=db.example.com

# Multiple key-values
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=db.example.com \
  --from-literal=DATABASE_PORT=5432 \
  --from-literal=LOG_LEVEL=info
```

#### Method 2: Imperative - From File

```bash
# From a single file (filename becomes key)
kubectl create configmap app-config --from-file=config.json

# From file with custom key
kubectl create configmap app-config --from-file=app-config=config.json

# From multiple files
kubectl create configmap app-config \
  --from-file=config.json \
  --from-file=settings.yaml
```

#### Method 3: Imperative - From Directory

```bash
# All files in directory become keys
kubectl create configmap app-config --from-directory=./config/
```

#### Method 4: Imperative - From Environment File

```bash
# .env file format: KEY=VALUE
cat > app.env << EOF
DATABASE_HOST=db.example.com
DATABASE_PORT=5432
LOG_LEVEL=info
EOF

kubectl create configmap app-config --from-env-file=app.env
```

#### Method 5: Declarative - YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
  labels:
    app: myapp
data:
  # Simple key-value pairs
  DATABASE_HOST: "db.example.com"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  
  # Multi-line configuration file
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      pool_size: 10
      timeout: 30s
    logging:
      level: info
      format: json
  
  # JSON configuration file
  config.json: |
    {
      "feature_flags": {
        "new_ui": true,
        "beta_features": false
      }
    }
```

### 1.4 Using ConfigMaps

#### Option 1: Environment Variables - Single Key

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-single-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_HOST                    # Env var name in container
      valueFrom:
        configMapKeyRef:
          name: app-config             # ConfigMap name
          key: DATABASE_HOST           # Key in ConfigMap
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
```

#### Option 2: Environment Variables - All Keys (envFrom)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-all-pod
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - configMapRef:
        name: app-config              # All keys become env vars
      prefix: APP_                    # Optional: prefix all vars
```

#### Option 3: Volume Mount - All Keys as Files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-all-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config          # Each key = file
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

#### Option 4: Volume Mount - Specific Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-specific-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: config.yaml
        path: application.yaml       # Rename the file
      - key: config.json
        path: settings/config.json   # Subdirectory
```

#### Option 5: Volume Mount - SubPath (Single File)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx.conf            # Only mount this file
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

### 1.5 ConfigMap Usage Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 ENVIRONMENT VARIABLES vs VOLUME MOUNTS                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ENVIRONMENT VARIABLES                                                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ✅ Simple to use                                              │   │
│   │   ✅ Works with any application                                 │   │
│   │   ✅ No file system complexity                                  │   │
│   │                                                                 │   │
│   │   ❌ NOT updated when ConfigMap changes                         │   │
│   │   ❌ Requires pod restart for updates                           │   │
│   │   ❌ Not suitable for large configs                             │   │
│   │   ❌ All processes see all env vars                             │   │
│   │                                                                 │   │
│   │   Best for: Simple values, feature flags, connection strings    │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   VOLUME MOUNTS                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ✅ Automatically updated when ConfigMap changes*              │   │
│   │   ✅ Works for large configuration files                        │   │
│   │   ✅ Maintains file structure                                   │   │
│   │   ✅ Can set file permissions                                   │   │
│   │                                                                 │   │
│   │   ❌ More complex setup                                         │   │
│   │   ❌ Application must watch for file changes                    │   │
│   │   ❌ subPath mounts are NOT updated                             │   │
│   │                                                                 │   │
│   │   Best for: Config files, complex configs, live updates        │   │
│   │                                                                 │   │
│   │   * Update delay: ~1 minute (kubelet sync period)              │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.6 Immutable ConfigMaps

Starting with Kubernetes 1.21, you can make ConfigMaps immutable for improved performance and security:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  API_ENDPOINT: "https://api.example.com"
immutable: true    # Cannot be changed after creation
```

**Benefits of Immutable ConfigMaps:**
- Protects from accidental updates
- Significantly reduces load on kube-apiserver (no watches)
- Faster propagation to kubelet
- Better for production

---

## 2. Secrets

### 2.1 What is a Secret?

A **Secret** is a Kubernetes object that stores sensitive data such as passwords, tokens, and keys. Secrets are similar to ConfigMaps but are intended for confidential data.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SECRET ANATOMY                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   apiVersion: v1                                                        │
│   kind: Secret                                                          │
│   metadata:                                                             │
│     name: my-secret                                                     │
│   type: Opaque                 ◀── Secret type                         │
│   data:                        ◀── Base64 encoded values               │
│     username: YWRtaW4=               # admin                           │
│     password: cGFzc3dvcmQxMjM=       # password123                     │
│   stringData:                  ◀── Plain text (auto-encoded)           │
│     api-key: "sk_live_xxxx"          # Encoded on apply                │
│                                                                         │
│   ⚠️  Note: Base64 is ENCODING, not ENCRYPTION!                        │
│       Anyone with access to the Secret can decode it.                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Secret Types

| Type | Description | Use Case |
|------|-------------|----------|
| `Opaque` | Arbitrary user-defined data | Generic secrets |
| `kubernetes.io/tls` | TLS certificate and key | HTTPS, mTLS |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials | Private registries |
| `kubernetes.io/service-account-token` | Service account token | Pod identity |
| `kubernetes.io/basic-auth` | Basic authentication | Username/password |
| `kubernetes.io/ssh-auth` | SSH private key | Git access, SSH |

### 2.3 Creating Secrets

#### Method 1: Imperative - From Literals

```bash
# Generic secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123

# View the secret (base64 encoded)
kubectl get secret db-credentials -o yaml
```

#### Method 2: Imperative - From Files

```bash
# Create files with credentials
echo -n 'admin' > ./username.txt
echo -n 'password123' > ./password.txt

# Create secret from files
kubectl create secret generic db-credentials \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt

# Clean up files
rm ./username.txt ./password.txt
```

#### Method 3: TLS Secret

```bash
# Create TLS secret from cert and key files
kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

#### Method 4: Docker Registry Secret

```bash
# For private Docker registries
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

#### Method 5: Declarative - YAML with data (base64)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Base64 encoded values
  username: YWRtaW4=           # echo -n 'admin' | base64
  password: cGFzc3dvcmQxMjM=   # echo -n 'password123' | base64
```

#### Method 6: Declarative - YAML with stringData (plain text)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  # Plain text - Kubernetes encodes to base64 automatically
  username: admin
  password: password123
  connection-string: "postgresql://admin:password123@db:5432/mydb"
```

### 2.4 Using Secrets

#### Option 1: Environment Variables - Single Key

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

#### Option 2: Environment Variables - All Keys

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-envfrom-pod
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - secretRef:
        name: db-credentials
      prefix: DB_              # Optional prefix
```

#### Option 3: Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true           # Best practice
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400        # Restrict permissions
```

#### Option 4: TLS Secret for Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: my-tls-secret    # Reference TLS secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

#### Option 5: Image Pull Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: registry.example.com/myapp:latest
  imagePullSecrets:
  - name: my-registry-secret    # Docker registry secret
```

### 2.5 Secret Security

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SECRET SECURITY LAYERS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   DEFAULT PROTECTION (Minimal):                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │   • Base64 encoding (NOT encryption!)                           │   │
│   │   • RBAC controls who can read secrets                          │   │
│   │   • Secrets only sent to nodes that need them                   │   │
│   │   • Stored in tmpfs on nodes (not written to disk)              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   RECOMMENDED ENHANCEMENTS:                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   1. ENCRYPTION AT REST                                         │   │
│   │      ┌─────────────────────────────────────────────────────┐   │   │
│   │      │  Enable etcd encryption:                             │   │   │
│   │      │  - Configure EncryptionConfiguration                 │   │   │
│   │      │  - Use AES-CBC or AES-GCM                           │   │   │
│   │      │  - Managed K8s (EKS/GKE/AKS) does this by default   │   │   │
│   │      └─────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   2. STRICT RBAC                                                │   │
│   │      ┌─────────────────────────────────────────────────────┐   │   │
│   │      │  Limit who can:                                      │   │   │
│   │      │  - get/list/watch secrets                           │   │   │
│   │      │  - create/update/delete secrets                     │   │   │
│   │      │  - exec into pods that have secrets                 │   │   │
│   │      └─────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   3. EXTERNAL SECRET MANAGEMENT                                 │   │
│   │      ┌─────────────────────────────────────────────────────┐   │   │
│   │      │  Use external systems:                               │   │   │
│   │      │  - HashiCorp Vault                                  │   │   │
│   │      │  - AWS Secrets Manager                              │   │   │
│   │      │  - Azure Key Vault                                  │   │   │
│   │      │  - Google Secret Manager                            │   │   │
│   │      │                                                      │   │   │
│   │      │  Operators:                                          │   │   │
│   │      │  - External Secrets Operator                        │   │   │
│   │      │  - Sealed Secrets                                   │   │   │
│   │      │  - Vault Secrets Operator                           │   │   │
│   │      └─────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ⚠️  REMEMBER: Base64 decoded easily!                                 │
│       echo "YWRtaW4=" | base64 -d  →  admin                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. ConfigMaps vs Secrets Comparison

### 3.1 Feature Comparison

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive config | Sensitive data |
| **Encoding** | Plain text | Base64 |
| **Encryption at Rest** | No (by default) | Supported |
| **Max Size** | 1 MiB | 1 MiB |
| **Update Propagation** | ~1 minute | ~1 minute |
| **tmpfs Storage** | No | Yes (on node) |
| **API Access** | Same RBAC | Often stricter |
| **Immutable Option** | Yes | Yes |

### 3.2 When to Use What

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGMAP vs SECRET DECISION                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Use ConfigMap for:                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  ✅ Application configuration files                             │   │
│   │  ✅ Environment-specific settings                               │   │
│   │  ✅ Feature flags                                               │   │
│   │  ✅ Connection pool sizes, timeouts                             │   │
│   │  ✅ Logging configuration                                       │   │
│   │  ✅ Nginx/Apache config files                                   │   │
│   │  ✅ Scripts to be mounted and executed                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Use Secret for:                                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  🔒 Database passwords                                          │   │
│   │  🔒 API keys and tokens                                         │   │
│   │  🔒 TLS certificates and private keys                           │   │
│   │  🔒 SSH keys                                                    │   │
│   │  🔒 OAuth credentials                                           │   │
│   │  🔒 Docker registry credentials                                 │   │
│   │  🔒 Encryption keys                                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ⚠️  If in doubt, use Secret - it's always safer!                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Working with ConfigMaps and Secrets

### 4.1 Base64 Encoding/Decoding

```bash
# Encode to base64
echo -n 'my-password' | base64
# Output: bXktcGFzc3dvcmQ=

# Decode from base64
echo 'bXktcGFzc3dvcmQ=' | base64 -d
# Output: my-password

# Encode file content
base64 -w 0 < certificate.pem

# Decode to file
echo 'encoded-content' | base64 -d > decoded-file
```

### 4.2 View ConfigMap/Secret Data

```bash
# View ConfigMap
kubectl get configmap my-config -o yaml

# View Secret (still base64 encoded in output)
kubectl get secret my-secret -o yaml

# Decode specific secret value
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# View all secret values decoded
kubectl get secret my-secret -o go-template='{{range $k,$v := .data}}{{$k}}: {{$v | base64decode}}{{"\n"}}{{end}}'
```

### 4.3 Update ConfigMap/Secret

```bash
# Edit directly
kubectl edit configmap my-config

# Replace from file
kubectl create configmap my-config --from-file=config.yaml --dry-run=client -o yaml | kubectl apply -f -

# Patch specific key
kubectl patch configmap my-config --type merge -p '{"data":{"LOG_LEVEL":"debug"}}'

# Delete and recreate
kubectl delete configmap my-config
kubectl create configmap my-config --from-literal=key=newvalue
```

### 4.4 Verify ConfigMap/Secret in Pod

```bash
# Check environment variables
kubectl exec my-pod -- env | grep DB

# Check mounted files
kubectl exec my-pod -- ls -la /etc/config

# Read mounted config file
kubectl exec my-pod -- cat /etc/config/config.yaml

# Check secret file permissions
kubectl exec my-pod -- ls -la /etc/secrets
```

---

## 5. Advanced Patterns

### 5.1 Projected Volumes with ConfigMap and Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: all-in-one
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - configMap:
          name: app-config
          items:
          - key: config.yaml
            path: config/app.yaml
      - secret:
          name: app-secret
          items:
          - key: api-key
            path: secrets/api-key
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
```

### 5.2 Init Container to Wait for Config

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wait-for-config
spec:
  initContainers:
  - name: wait-for-config
    image: busybox
    command: ['sh', '-c', 'until [ -f /etc/config/ready ]; do echo waiting; sleep 2; done']
    volumeMounts:
    - name: config
      mountPath: /etc/config
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```

### 5.3 Sidecar for Dynamic Config Reload

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-reloader
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config
      mountPath: /etc/config
  - name: reloader
    image: jimmidyson/configmap-reload
    args:
    - --volume-dir=/etc/config
    - --webhook-url=http://localhost:8080/-/reload
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

### 5.4 Per-Environment Configuration

```yaml
# base-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: myapp
  LOG_FORMAT: json
---
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: myapp
  LOG_FORMAT: json
  LOG_LEVEL: debug
  DATABASE_HOST: dev-db.internal
---
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: myapp
  LOG_FORMAT: json
  LOG_LEVEL: warn
  DATABASE_HOST: prod-db.internal
```

---

## 6. External Secrets Management

### 6.1 External Secrets Pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   EXTERNAL SECRETS ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                  External Secret Store                          │   │
│   │   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │   │
│   │   │  Vault    │  │ AWS SM    │  │ Azure KV  │  │ GCP SM    │   │   │
│   │   │           │  │           │  │           │  │           │   │   │
│   │   └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘   │   │
│   │         │              │              │              │         │   │
│   └─────────┼──────────────┼──────────────┼──────────────┼─────────┘   │
│             │              │              │              │             │
│             └──────────────┴──────────────┴──────────────┘             │
│                                    │                                    │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              External Secrets Operator                          │   │
│   │                                                                 │   │
│   │   Watches ExternalSecret CRDs                                   │   │
│   │   Fetches secrets from external stores                          │   │
│   │   Creates Kubernetes Secrets                                    │   │
│   │   Keeps secrets in sync                                         │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              Kubernetes Secret                                  │   │
│   │                                                                 │   │
│   │   Automatically created and synced                              │   │
│   │   Used by Pods like any other Secret                            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Benefits:                                                             │
│   ✅ Secrets never stored in Git                                       │
│   ✅ Centralized secret management                                     │
│   ✅ Audit logging in external store                                   │
│   ✅ Rotation handled externally                                       │
│   ✅ Works with existing secret infrastructure                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 External Secrets Operator Example

```yaml
# SecretStore - defines how to connect to external store
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
# ExternalSecret - defines what to fetch
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: db-credentials        # K8s Secret to create
    creationPolicy: Owner
  data:
  - secretKey: username         # Key in K8s Secret
    remoteRef:
      key: prod/database        # Secret name in AWS
      property: username        # JSON key in secret
  - secretKey: password
    remoteRef:
      key: prod/database
      property: password
```

### 6.3 Sealed Secrets (GitOps-Friendly)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SEALED SECRETS WORKFLOW                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1. Create Secret normally (locally)                                   │
│      ┌────────────────────────────────────┐                            │
│      │  kubectl create secret generic ... │                            │
│      │  --dry-run=client -o yaml          │                            │
│      └───────────────┬────────────────────┘                            │
│                      │                                                  │
│                      ▼                                                  │
│   2. Encrypt with kubeseal                                              │
│      ┌────────────────────────────────────┐                            │
│      │  kubeseal --format yaml <secret.yaml│                           │
│      │          > sealed-secret.yaml       │                            │
│      └───────────────┬────────────────────┘                            │
│                      │                                                  │
│                      ▼                                                  │
│   3. Commit SealedSecret to Git                                         │
│      ┌────────────────────────────────────┐                            │
│      │  git add sealed-secret.yaml        │  Safe! Encrypted.          │
│      │  git commit -m "Add db secret"     │                            │
│      └───────────────┬────────────────────┘                            │
│                      │                                                  │
│                      ▼                                                  │
│   4. Controller decrypts in cluster                                     │
│      ┌────────────────────────────────────┐                            │
│      │  SealedSecret → Secret             │                            │
│      │  (Only controller has private key) │                            │
│      └────────────────────────────────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```yaml
# Sealed Secret YAML (safe to commit)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
  template:
    type: Opaque
    metadata:
      name: db-credentials
```

---

## 7. Best Practices

### 7.1 ConfigMap Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGMAP BEST PRACTICES                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Use descriptive names (app-name-config, not cm1)             │   │
│   │  • Include environment in name if needed (app-config-prod)      │   │
│   │  • Use immutable: true for production stability                 │   │
│   │  • Version config with annotations for rollback                 │   │
│   │  • Use separate ConfigMaps for different purposes               │   │
│   │  • Keep ConfigMaps small and focused                            │   │
│   │  • Document expected keys and values                            │   │
│   │  • Use volume mounts for config files that apps watch           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Store sensitive data in ConfigMaps                           │   │
│   │  • Create ConfigMaps larger than 1MB                            │   │
│   │  • Depend on immediate updates (there's a delay)                │   │
│   │  • Use subPath if you need automatic updates                    │   │
│   │  • Assume env vars update without restart                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Secret Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SECRET BEST PRACTICES                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ DO:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Enable encryption at rest in etcd                            │   │
│   │  • Use RBAC to limit secret access                              │   │
│   │  • Mount secrets as volumes with restrictive permissions        │   │
│   │  • Use external secret management (Vault, AWS SM) in prod       │   │
│   │  • Rotate secrets regularly                                     │   │
│   │  • Use immutable secrets when possible                          │   │
│   │  • Audit secret access                                          │   │
│   │  • Use readOnly: true for secret volume mounts                  │   │
│   │  • Set defaultMode: 0400 for secret files                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ DON'T:                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  • Commit secrets to Git (even encrypted, unless Sealed)        │   │
│   │  • Log secret values                                            │   │
│   │  • Pass secrets via command-line arguments                      │   │
│   │  • Assume base64 = encrypted (it's NOT!)                        │   │
│   │  • Give broad RBAC access to secrets                            │   │
│   │  • Use same secrets across environments                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Command Reference

### ConfigMap Commands

```bash
# Create
kubectl create configmap my-config --from-literal=key=value
kubectl create configmap my-config --from-file=config.yaml
kubectl create configmap my-config --from-file=./configs/
kubectl create configmap my-config --from-env-file=app.env

# List
kubectl get configmaps
kubectl get cm                        # Short form

# View
kubectl describe configmap my-config
kubectl get configmap my-config -o yaml

# Edit
kubectl edit configmap my-config

# Delete
kubectl delete configmap my-config

# View specific key
kubectl get configmap my-config -o jsonpath='{.data.key}'
```

### Secret Commands

```bash
# Create
kubectl create secret generic my-secret --from-literal=password=secret123
kubectl create secret generic my-secret --from-file=ssh-key=~/.ssh/id_rsa
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass

# List
kubectl get secrets
kubectl get secret                    # Short form

# View (still encoded)
kubectl describe secret my-secret
kubectl get secret my-secret -o yaml

# Decode specific key
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Delete
kubectl delete secret my-secret
```

---

## 9. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| Create ConfigMap/Secret | 🔴 HIGH | Imperative and declarative |
| Use as env vars | 🔴 HIGH | `valueFrom`, `envFrom` |
| Mount as volumes | 🔴 HIGH | `volumeMounts`, `volumes` |
| Decode secrets | 🟡 MEDIUM | `base64 -d` |
| Update ConfigMap | 🟡 MEDIUM | Edit, patch, replace |

### Quick Reference for Exam

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  KEY: value

# Secret
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  PASSWORD: secret123

# Use as env var
env:
- name: MY_VAR
  valueFrom:
    configMapKeyRef:
      name: my-config
      key: KEY

# Use as volume
volumes:
- name: config-vol
  configMap:
    name: my-config
```

### Common Exam Mistakes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA MISTAKES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ❌ Mistake 1: Wrong key reference                                     │
│      configMapKeyRef.key must match key in ConfigMap.data              │
│                                                                         │
│   ❌ Mistake 2: Forgetting base64 encoding in YAML                      │
│      Use stringData (plain) instead of data (base64) for easier YAML   │
│                                                                         │
│   ❌ Mistake 3: ConfigMap/Secret not in same namespace                  │
│      Pod and ConfigMap/Secret must be in same namespace                │
│                                                                         │
│   ❌ Mistake 4: Expecting immediate env var updates                     │
│      Env vars require pod restart; volume mounts update automatically  │
│                                                                         │
│   ❌ Mistake 5: Using subPath for dynamic configs                       │
│      subPath mounts do NOT update when ConfigMap changes               │
│                                                                         │
│   ❌ Mistake 6: Wrong Secret type                                       │
│      TLS secrets need type: kubernetes.io/tls with tls.crt & tls.key   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Docker to Kubernetes Mapping

| Docker Concept | Kubernetes Equivalent |
|---------------|----------------------|
| `docker run -e VAR=value` | ConfigMap/Secret as env var |
| `docker run --env-file .env` | ConfigMap/Secret with envFrom |
| `docker run -v config.json:/app/config.json` | ConfigMap/Secret volume mount |
| Docker Compose `.env` file | ConfigMap |
| Docker Secrets (Swarm) | Kubernetes Secrets |
| Environment variables in Dockerfile | ConfigMap/Secret at runtime |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  CONFIGURATION DECISION FLOWCHART                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Is the data sensitive?                                                │
│       │                                                                 │
│       ├──▶ YES → Use Secret                                            │
│       │           │                                                     │
│       │           ├──▶ Need external management? → External Secrets    │
│       │           │                                                     │
│       │           └──▶ Need GitOps? → Sealed Secrets                   │
│       │                                                                 │
│       └──▶ NO → Use ConfigMap                                          │
│                   │                                                     │
│                   ├──▶ Simple values? → Environment variables          │
│                   │                                                     │
│                   └──▶ Config files? → Volume mounts                   │
│                                                                         │
│   Need live updates?                                                    │
│       │                                                                 │
│       ├──▶ YES → Volume mount (not subPath) + app file watching        │
│       │                                                                 │
│       └──▶ NO → Environment variables (simpler)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **RBAC (Role-Based Access Control)** - Controlling who can do what
- Roles, ClusterRoles, RoleBindings, ClusterRoleBindings
- Service Accounts
- Authentication vs Authorization

---

**Chapter 17 Complete! 🎉**

You now understand:
- ConfigMaps for non-sensitive configuration
- Secrets for sensitive data
- Multiple ways to create and consume configs
- Environment variables vs volume mounts
- Security best practices
- External secrets management patterns
- CKA exam preparation

