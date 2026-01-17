# Chapter 19: Security Contexts - Pod and Container Security

## Introduction

By default, containers run with more privileges than they need. A container could run as root, access any file in its filesystem, and use powerful Linux capabilities. **Security Contexts** let you lock down pods and containers to follow the principle of least privilege.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE CONTAINER SECURITY PROBLEM                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Default Container (Risky):                                            │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                     CONTAINER                           │   │   │
│   │   │                                                         │   │   │
│   │   │   • Running as root (UID 0)        ← DANGEROUS         │   │   │
│   │   │   • Full filesystem write access   ← DANGEROUS         │   │   │
│   │   │   • All Linux capabilities         ← DANGEROUS         │   │   │
│   │   │   • Can escalate privileges        ← DANGEROUS         │   │   │
│   │   │   • No isolation profiles          ← DANGEROUS         │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   If container is compromised → attacker has root access!       │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Hardened Container (Secure):                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                     CONTAINER                           │   │   │
│   │   │                                                         │   │   │
│   │   │   • Running as non-root (UID 1000) ← SAFE              │   │   │
│   │   │   • Read-only root filesystem      ← SAFE              │   │   │
│   │   │   • Minimal capabilities           ← SAFE              │   │   │
│   │   │   • No privilege escalation        ← SAFE              │   │   │
│   │   │   • Seccomp profile applied        ← SAFE              │   │   │
│   │   │                                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   │   If container is compromised → attacker is limited!           │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. What is a Security Context?

A **Security Context** defines privilege and access control settings for a Pod or Container. It specifies:

- User and group IDs to run as
- Linux capabilities to add or drop
- Whether to run as privileged
- Filesystem permissions
- SELinux/AppArmor/Seccomp profiles

### 1.1 Pod-Level vs Container-Level

```
┌─────────────────────────────────────────────────────────────────────────┐
│                POD-LEVEL vs CONTAINER-LEVEL SECURITY                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                           POD                                   │   │
│   │  spec:                                                          │   │
│   │    securityContext:           ◀── POD-LEVEL                    │   │
│   │      runAsUser: 1000              (applies to all containers)   │   │
│   │      fsGroup: 2000                                              │   │
│   │                                                                 │   │
│   │  ┌──────────────────────┐  ┌──────────────────────┐            │   │
│   │  │    Container A       │  │    Container B       │            │   │
│   │  │                      │  │                      │            │   │
│   │  │  securityContext:    │  │  securityContext:    │            │   │
│   │  │    readOnlyRoot: ◀───┼──│    runAsUser: 2000 ◀─────────────│   │
│   │  │      Filesystem:     │  │    (overrides pod)   │  CONTAINER-│   │
│   │  │      true            │  │                      │  LEVEL     │   │
│   │  └──────────────────────┘  └──────────────────────┘            │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Precedence: Container-level overrides Pod-level                       │
│                                                                         │
│   Pod-Level Settings:               Container-Level Settings:           │
│   • runAsUser                       • runAsUser (overrides)             │
│   • runAsGroup                      • runAsGroup (overrides)            │
│   • runAsNonRoot                    • runAsNonRoot (overrides)          │
│   • fsGroup                         • readOnlyRootFilesystem            │
│   • fsGroupChangePolicy             • allowPrivilegeEscalation          │
│   • supplementalGroups              • capabilities                      │
│   • sysctls                         • privileged                        │
│   • seccompProfile                  • seccompProfile (overrides)        │
│   • seLinuxOptions                  • seLinuxOptions (overrides)        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Running as Non-Root

### 2.1 Why Run as Non-Root?

Running as root inside a container is dangerous because:
- Container breakout vulnerabilities give attacker root on host
- Malicious code runs with maximum privileges
- Can modify system files and escalate further

### 2.2 runAsUser

Specifies the UID to run the container process as:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-user
spec:
  securityContext:
    runAsUser: 1000           # Run all containers as UID 1000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'id && sleep 3600']
```

```bash
# Verify the user
kubectl exec run-as-user -- id
# Output: uid=1000 gid=0(root) groups=0(root)
```

### 2.3 runAsGroup

Specifies the primary GID:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-group
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000          # Primary group
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'id && sleep 3600']
```

```bash
# Output: uid=1000 gid=3000 groups=3000
```

### 2.4 runAsNonRoot

**Enforces** that the container must run as non-root:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: must-run-as-non-root
spec:
  securityContext:
    runAsNonRoot: true        # Container must NOT run as root
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'sleep 3600']
    securityContext:
      runAsUser: 1000         # Satisfy the requirement
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    runAsNonRoot BEHAVIOR                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   runAsNonRoot: true                                                    │
│       │                                                                 │
│       ▼                                                                 │
│   Is runAsUser specified?                                               │
│       │                                                                 │
│       ├──▶ YES: Is it 0?                                               │
│       │         │                                                       │
│       │         ├──▶ YES → Pod fails to start                          │
│       │         │         "container has runAsNonRoot but UID 0"       │
│       │         │                                                       │
│       │         └──▶ NO → Pod starts successfully                      │
│       │                                                                 │
│       └──▶ NO: Check image's USER directive                            │
│                 │                                                       │
│                 ├──▶ Image runs as root → Pod fails to start           │
│                 │                                                       │
│                 └──▶ Image runs as non-root → Pod starts               │
│                                                                         │
│   Best Practice: Always set runAsUser explicitly when using            │
│                  runAsNonRoot: true                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 supplementalGroups

Additional group memberships:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: supplemental-groups
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    supplementalGroups: [4000, 5000]    # Additional groups
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'id && sleep 3600']
```

```bash
# Output: uid=1000 gid=3000 groups=3000,4000,5000
```

---

## 3. Filesystem Security

### 3.1 fsGroup

Sets the group ownership of mounted volumes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fs-group-demo
spec:
  securityContext:
    fsGroup: 2000             # Volume files owned by GID 2000
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls -la /data && sleep 3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        fsGroup BEHAVIOR                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Without fsGroup:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  /data/                                                         │   │
│   │  -rw-r--r-- root root  file1.txt                               │   │
│   │  -rw-r--r-- root root  file2.txt                               │   │
│   │                                                                 │   │
│   │  Container user (UID 1000) can't write!                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With fsGroup: 2000:                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  /data/                                                         │   │
│   │  -rw-rw---- root 2000  file1.txt                               │   │
│   │  -rw-rw---- root 2000  file2.txt                               │   │
│   │                                                                 │   │
│   │  Container user (member of GID 2000) CAN write!                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   What fsGroup does:                                                    │
│   1. Adds fsGroup to container's supplemental groups                   │
│   2. Changes volume file ownership to fsGroup                          │
│   3. Sets setgid bit on volume directories                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 fsGroupChangePolicy

Controls how volume permissions are changed (Kubernetes 1.20+):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fs-group-policy
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"   # or "Always"
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `Always` | Always change ownership/permissions | Strict security |
| `OnRootMismatch` | Only change if root ownership differs | Large volumes (faster) |

### 3.3 readOnlyRootFilesystem

Prevents writes to the container's root filesystem:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: read-only-fs
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true    # Can't write to /
    volumeMounts:
    - name: tmp
      mountPath: /tmp                  # Writable temp directory
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  readOnlyRootFilesystem PATTERN                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      CONTAINER                                  │   │
│   │                                                                 │   │
│   │   /  (root)  ← READ-ONLY (immutable)                           │   │
│   │   ├── app/   ← READ-ONLY                                       │   │
│   │   ├── etc/   ← READ-ONLY                                       │   │
│   │   ├── usr/   ← READ-ONLY                                       │   │
│   │   │                                                             │   │
│   │   ├── tmp/   ← WRITABLE (emptyDir mounted)                     │   │
│   │   ├── var/                                                      │   │
│   │   │   ├── cache/nginx/ ← WRITABLE (emptyDir)                   │   │
│   │   │   └── run/         ← WRITABLE (emptyDir)                   │   │
│   │   │                                                             │   │
│   │   └── data/  ← WRITABLE (PVC mounted)                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Benefits:                                                             │
│   • Prevents malware from modifying container                          │
│   • Attacker can't install backdoors                                   │
│   • Forces proper use of volumes for state                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Linux Capabilities

### 4.1 What are Capabilities?

Linux capabilities divide root's powers into smaller units. Instead of giving full root access, you can grant specific privileges.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     LINUX CAPABILITIES                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Traditional Linux:                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Root (UID 0)           │  Non-root                           │   │
│   │   ────────────────────   │  ────────────────────               │   │
│   │   ALL privileges         │  NO privileges                      │   │
│   │   Can do EVERYTHING      │  Can do very little                 │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   With Capabilities:                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Root's power broken into ~40 capabilities:                    │   │
│   │                                                                 │   │
│   │   CAP_NET_BIND_SERVICE  ─  Bind to ports < 1024                │   │
│   │   CAP_NET_ADMIN         ─  Network configuration               │   │
│   │   CAP_SYS_ADMIN         ─  System administration (DANGEROUS)   │   │
│   │   CAP_CHOWN             ─  Change file ownership               │   │
│   │   CAP_SETUID            ─  Change UID                          │   │
│   │   CAP_SETGID            ─  Change GID                          │   │
│   │   CAP_KILL              ─  Send signals to any process         │   │
│   │   CAP_NET_RAW           ─  Use raw sockets                     │   │
│   │   CAP_SYS_PTRACE        ─  Trace processes                     │   │
│   │   CAP_MKNOD             ─  Create device files                 │   │
│   │   CAP_AUDIT_WRITE       ─  Write to audit log                  │   │
│   │   ... and more                                                  │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Docker Default Capabilities (granted by default):                     │
│   CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER, CAP_FSETID,                │
│   CAP_KILL, CAP_SETGID, CAP_SETUID, CAP_SETPCAP,                      │
│   CAP_NET_BIND_SERVICE, CAP_NET_RAW, CAP_SYS_CHROOT,                  │
│   CAP_MKNOD, CAP_AUDIT_WRITE, CAP_SETFCAP                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Dropping Capabilities

Remove unnecessary capabilities:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: drop-caps
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL                 # Drop ALL capabilities first
        add:
        - NET_BIND_SERVICE    # Then add only what's needed
```

### 4.3 Adding Capabilities

Add specific capabilities (use sparingly):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: add-caps
spec:
  containers:
  - name: app
    image: myapp
    securityContext:
      capabilities:
        add:
        - NET_ADMIN           # For network tools
        - SYS_TIME            # For time sync
```

### 4.4 Common Capabilities Reference

| Capability | Description | When Needed |
|------------|-------------|-------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 | Web servers on 80/443 |
| `NET_ADMIN` | Network configuration | Network debugging, VPN |
| `NET_RAW` | Raw sockets (ping, etc.) | Network debugging |
| `SYS_ADMIN` | Various admin ops | **AVOID** - too broad |
| `SYS_PTRACE` | Process tracing | Debugging, profiling |
| `SYS_TIME` | Modify system time | Time sync daemons |
| `CHOWN` | Change file ownership | File management |
| `SETUID/SETGID` | Change UID/GID | User switching |
| `KILL` | Send signals | Process management |

### 4.5 Recommended Capability Configuration

```yaml
# Secure baseline: drop all, add only necessary
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE    # Only if needed for port < 1024
```

---

## 5. Privilege Escalation

### 5.1 allowPrivilegeEscalation

Controls whether a process can gain more privileges than its parent:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-privilege-escalation
spec:
  containers:
  - name: app
    image: myapp
    securityContext:
      allowPrivilegeEscalation: false   # Recommended
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  allowPrivilegeEscalation                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   allowPrivilegeEscalation: true (default)                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Process (UID 1000)                                            │   │
│   │       │                                                         │   │
│   │       │  Executes setuid binary                                 │   │
│   │       ▼                                                         │   │
│   │   Process (UID 0)  ← CAN become root!                          │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   allowPrivilegeEscalation: false (secure)                              │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Process (UID 1000)                                            │   │
│   │       │                                                         │   │
│   │       │  Executes setuid binary                                 │   │
│   │       ▼                                                         │   │
│   │   Process (UID 1000)  ← Stays as original user                 │   │
│   │                       (setuid bit ignored)                      │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   What it prevents:                                                     │
│   • setuid binaries (like sudo, su)                                    │
│   • no_new_privs Linux kernel flag                                     │
│   • Process can't gain capabilities not in parent's set               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 privileged

Gives container full host access (**DANGEROUS**):

```yaml
# DON'T do this unless absolutely necessary!
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: app
    image: myapp
    securityContext:
      privileged: true        # Full host access - AVOID!
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ⚠️  PRIVILEGED CONTAINERS                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   privileged: true gives the container:                                 │
│                                                                         │
│   • ALL Linux capabilities                                              │
│   • Access to ALL host devices (/dev/*)                                │
│   • Can load kernel modules                                             │
│   • Can modify host's iptables                                         │
│   • Essentially root on the host                                        │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                 │   │
│   │   Privileged Container                                          │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  "I AM the host"                                        │   │   │
│   │   │                                                         │   │   │
│   │   │  Can access:                                            │   │   │
│   │   │  • /dev/sda (host disk)                                │   │   │
│   │   │  • /dev/mem (host memory)                              │   │   │
│   │   │  • Can escape container trivially                      │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Legitimate use cases (rare):                                          │
│   • Container runtimes (Docker-in-Docker)                              │
│   • System debugging tools                                              │
│   • Low-level storage drivers                                           │
│                                                                         │
│   ⚠️  NEVER use privileged: true for normal applications!             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Seccomp Profiles

### 6.1 What is Seccomp?

**Seccomp** (Secure Computing Mode) limits the system calls a container can make. This reduces the attack surface by blocking unused syscalls.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SECCOMP OVERVIEW                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      CONTAINER                                  │   │
│   │                                                                 │   │
│   │   Application Code                                              │   │
│   │        │                                                        │   │
│   │        │  Makes syscall (read, write, open, etc.)              │   │
│   │        ▼                                                        │   │
│   │   ┌────────────────────────────────────────────────────────┐   │   │
│   │   │              SECCOMP FILTER                            │   │   │
│   │   │                                                        │   │   │
│   │   │  Is this syscall allowed?                              │   │   │
│   │   │                                                        │   │   │
│   │   │  ├── read()   → ALLOW                                  │   │   │
│   │   │  ├── write()  → ALLOW                                  │   │   │
│   │   │  ├── open()   → ALLOW                                  │   │   │
│   │   │  ├── mount()  → BLOCK!                                 │   │   │
│   │   │  ├── ptrace() → BLOCK!                                 │   │   │
│   │   │  └── ...                                               │   │   │
│   │   │                                                        │   │   │
│   │   └────────────────────────────────────────────────────────┘   │   │
│   │        │                                                        │   │
│   │        ▼                                                        │   │
│   │   Linux Kernel                                                  │   │
│   │                                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Without Seccomp: ~450+ syscalls available (huge attack surface)      │
│   With Seccomp: ~60 syscalls allowed (minimal attack surface)          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Seccomp Profile Types

| Type | Description |
|------|-------------|
| `RuntimeDefault` | Container runtime's default profile |
| `Localhost` | Custom profile from node's filesystem |
| `Unconfined` | No restrictions (**insecure**) |

### 6.3 Using RuntimeDefault Profile

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-default
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault      # Use container runtime's default
  containers:
  - name: app
    image: myapp
```

### 6.4 Using Custom Seccomp Profile

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-custom
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-profile.json   # On node
  containers:
  - name: app
    image: myapp
```

### 6.5 Container-Level Seccomp

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-container
spec:
  containers:
  - name: app
    image: myapp
    securityContext:
      seccompProfile:
        type: RuntimeDefault
```

---

## 7. AppArmor

### 7.1 What is AppArmor?

**AppArmor** is a Linux Security Module that restricts programs' capabilities with per-program profiles.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    # Container name: profile name
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
spec:
  containers:
  - name: app
    image: myapp
```

### 7.2 AppArmor Profile Options

| Profile | Description |
|---------|-------------|
| `runtime/default` | Container runtime's default profile |
| `localhost/<profile>` | Custom profile loaded on node |
| `unconfined` | No AppArmor restrictions |

---

## 8. SELinux

### 8.1 What is SELinux?

**SELinux** (Security-Enhanced Linux) provides mandatory access control using security labels.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selinux-pod
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"
      user: "system_u"
      role: "system_r"
      type: "container_t"
  containers:
  - name: app
    image: myapp
```

### 8.2 SELinux Options

| Option | Description |
|--------|-------------|
| `user` | SELinux user label |
| `role` | SELinux role label |
| `type` | SELinux type label |
| `level` | SELinux level (MCS/MLS) |

---

## 9. Complete Secure Pod Example

### 9.1 Maximum Security Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security
  securityContext:
    runAsNonRoot: true              # Must run as non-root
    runAsUser: 1000                 # Specific UID
    runAsGroup: 3000                # Specific GID
    fsGroup: 2000                   # Volume group ownership
    seccompProfile:
      type: RuntimeDefault          # Seccomp filtering
  
  # Don't mount service account token
  automountServiceAccountToken: false
  
  containers:
  - name: app
    image: myapp:latest
    
    # Container-level security
    securityContext:
      allowPrivilegeEscalation: false   # No privilege escalation
      readOnlyRootFilesystem: true      # Immutable container
      capabilities:
        drop:
        - ALL                           # Drop all capabilities
        # add:
        # - NET_BIND_SERVICE            # Add only if needed
    
    # Resource limits (also security-related)
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
    
    # Writable directories as volumes
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### 9.2 Security Context Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   SECURITY CONTEXT CHECKLIST                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ✅ Essential (Always Do):                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  □ runAsNonRoot: true                                           │   │
│   │  □ runAsUser: <non-zero UID>                                    │   │
│   │  □ allowPrivilegeEscalation: false                              │   │
│   │  □ readOnlyRootFilesystem: true                                 │   │
│   │  □ capabilities.drop: [ALL]                                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   🔒 Recommended (Production):                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  □ seccompProfile.type: RuntimeDefault                          │   │
│   │  □ automountServiceAccountToken: false (if no API access)      │   │
│   │  □ fsGroup for volume permissions                               │   │
│   │  □ Resource limits to prevent DoS                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   ❌ Avoid (Unless Absolutely Necessary):                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  □ privileged: true                                             │   │
│   │  □ capabilities.add: [SYS_ADMIN]                               │   │
│   │  □ hostNetwork: true                                            │   │
│   │  □ hostPID: true                                                │   │
│   │  □ hostIPC: true                                                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Host Namespaces

### 10.1 hostNetwork

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-network-pod
spec:
  hostNetwork: true           # Use host's network namespace
  containers:                 # Pod sees host's network interfaces
  - name: app
    image: myapp
```

**Use case:** Monitoring tools, CNI plugins
**Risk:** Pod can see all host network traffic

### 10.2 hostPID

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-pid-pod
spec:
  hostPID: true               # Use host's PID namespace
  containers:                 # Pod can see all host processes
  - name: app
    image: myapp
```

**Use case:** Debugging, process monitoring
**Risk:** Can kill host processes

### 10.3 hostIPC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-ipc-pod
spec:
  hostIPC: true               # Use host's IPC namespace
  containers:                 # Pod can access host shared memory
  - name: app
    image: myapp
```

**Use case:** Legacy applications using shared memory
**Risk:** Can access other containers' IPC

---

## 11. Debugging Security Contexts

### 11.1 Check Running User

```bash
# See what user the container is running as
kubectl exec my-pod -- id

# Check running processes
kubectl exec my-pod -- ps aux

# Check capabilities
kubectl exec my-pod -- cat /proc/1/status | grep Cap
```

### 11.2 Verify Filesystem

```bash
# Check if filesystem is read-only
kubectl exec my-pod -- touch /test-file
# Should fail if readOnlyRootFilesystem: true

# Check volume permissions
kubectl exec my-pod -- ls -la /data
```

### 11.3 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Pod won't start | runAsNonRoot but image runs as root | Set runAsUser explicitly |
| Permission denied on volume | Wrong fsGroup | Set correct fsGroup |
| Can't bind to port 80 | No NET_BIND_SERVICE cap | Add capability or use port > 1024 |
| App can't write | readOnlyRootFilesystem | Mount writable volumes |

---

## 12. Command Reference

```bash
# View pod security context
kubectl get pod my-pod -o yaml | grep -A20 securityContext

# Check running user
kubectl exec my-pod -- id

# Check capabilities
kubectl exec my-pod -- cat /proc/1/status | grep Cap

# Decode capabilities
capsh --decode=00000000a80425fb

# Check seccomp status
kubectl exec my-pod -- cat /proc/1/status | grep Seccomp

# Check AppArmor profile
kubectl exec my-pod -- cat /proc/1/attr/current
```

---

## 13. CKA Exam Tips

### High-Priority Topics

| Topic | CKA Weight | Key Skills |
|-------|------------|------------|
| runAsUser/runAsNonRoot | 🔴 HIGH | Set user ID |
| readOnlyRootFilesystem | 🔴 HIGH | Make container immutable |
| capabilities | 🟡 MEDIUM | Drop ALL, add specific |
| allowPrivilegeEscalation | 🟡 MEDIUM | Prevent privilege gain |
| fsGroup | 🟡 MEDIUM | Volume permissions |

### Quick Reference for Exam

```yaml
# Secure container
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

### Common Exam Scenarios

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMON CKA SCENARIOS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Scenario 1: "Make pod run as user 1000"                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  spec:                                                          │   │
│   │    securityContext:                                             │   │
│   │      runAsUser: 1000                                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 2: "Ensure pod cannot run as root"                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  spec:                                                          │   │
│   │    securityContext:                                             │   │
│   │      runAsNonRoot: true                                         │   │
│   │      runAsUser: 1000                                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 3: "Drop all capabilities except NET_BIND_SERVICE"          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  securityContext:                                               │   │
│   │    capabilities:                                                │   │
│   │      drop: ["ALL"]                                              │   │
│   │      add: ["NET_BIND_SERVICE"]                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   Scenario 4: "Make container filesystem read-only"                     │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  securityContext:                                               │   │
│   │    readOnlyRootFilesystem: true                                 │   │
│   │  volumeMounts:                                                  │   │
│   │  - name: tmp                                                    │   │
│   │    mountPath: /tmp                                              │   │
│   │  volumes:                                                       │   │
│   │  - name: tmp                                                    │   │
│   │    emptyDir: {}                                                 │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Docker to Kubernetes Mapping

| Docker Flag | Kubernetes Equivalent |
|-------------|----------------------|
| `--user 1000:1000` | `runAsUser`, `runAsGroup` |
| `--read-only` | `readOnlyRootFilesystem: true` |
| `--cap-drop ALL` | `capabilities.drop: ["ALL"]` |
| `--cap-add NET_BIND_SERVICE` | `capabilities.add: ["NET_BIND_SERVICE"]` |
| `--privileged` | `privileged: true` |
| `--security-opt no-new-privileges` | `allowPrivilegeEscalation: false` |
| `--security-opt seccomp=...` | `seccompProfile` |
| `--network host` | `hostNetwork: true` |
| `--pid host` | `hostPID: true` |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  SECURITY CONTEXT DECISION TREE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Start with secure defaults:                                           │
│                                                                         │
│   1. User/Group                                                         │
│      ├── runAsNonRoot: true                                            │
│      ├── runAsUser: <non-zero>                                         │
│      └── runAsGroup: <group-id>                                        │
│                                                                         │
│   2. Filesystem                                                         │
│      ├── readOnlyRootFilesystem: true                                  │
│      ├── fsGroup for volume permissions                                │
│      └── Mount emptyDir for writable paths                             │
│                                                                         │
│   3. Privileges                                                         │
│      ├── allowPrivilegeEscalation: false                               │
│      ├── privileged: false (never true!)                               │
│      └── capabilities:                                                  │
│          ├── drop: [ALL]                                               │
│          └── add: [only what's needed]                                 │
│                                                                         │
│   4. Additional Hardening                                               │
│      ├── seccompProfile: RuntimeDefault                                │
│      ├── AppArmor profile                                              │
│      └── automountServiceAccountToken: false                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover:
- **Pod Security Standards** - Cluster-wide security enforcement
- Pod Security Admission
- Restricted, Baseline, and Privileged profiles
- Migrating from PodSecurityPolicy

---

**Chapter 19 Complete! 🎉**

You now understand:
- Pod-level vs Container-level security contexts
- Running as non-root (runAsUser, runAsNonRoot)
- Filesystem security (readOnlyRootFilesystem, fsGroup)
- Linux capabilities (drop/add)
- Privilege escalation prevention
- Seccomp profiles
- AppArmor and SELinux basics
- Host namespaces
- Complete secure pod configuration
- CKA exam preparation

