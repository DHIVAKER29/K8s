# ☸️ Chapter 11: Jobs & CronJobs

> Running batch processing and scheduled tasks in Kubernetes - workloads that run to completion.

---

## 📚 Table of Contents

1. [What are Jobs?](#-what-are-jobs)
2. [Jobs vs Other Workloads](#-jobs-vs-other-workloads)
3. [Job Types](#-job-types)
4. [Creating Jobs](#-creating-jobs)
5. [Job Completions and Parallelism](#-job-completions-and-parallelism)
6. [Failure Handling](#-failure-handling)
7. [Job Cleanup](#-job-cleanup)
8. [CronJobs](#-cronjobs)
9. [Cron Syntax](#-cron-syntax)
10. [CronJob Configuration](#-cronjob-configuration)
11. [Common Use Cases](#-common-use-cases)
12. [Operations](#-operations)
13. [Troubleshooting](#-troubleshooting)
14. [CKA Exam Tips](#-cka-exam-tips)
15. [Summary](#-summary)

---

## 📖 What are Jobs?

### Definition

> **Job** creates one or more Pods and ensures that a specified number of them successfully terminate. Jobs track successful completions, and when a specified number is reached, the Job is complete.

### Key Concept

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           WHAT IS A JOB?                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Job = "Run this task until it completes successfully"                              │
│                                                                                      │
│  Regular Pod:                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Pod runs continuously (restartPolicy: Always)                              │   │
│  │  If it exits, it restarts                                                   │   │
│  │  Never "completes"                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Job-managed Pod:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Pod runs until successful exit (exit code 0)                               │   │
│  │  If it fails, Job retries (up to backoffLimit)                             │   │
│  │  Once successful, Job is "Complete"                                         │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Timeline:                                                                          │
│                                                                                      │
│  Job Created          Pod Runs           Pod Exits (0)      Job Complete           │
│       │                  │                    │                  │                  │
│       ▼                  ▼                    ▼                  ▼                  │
│  ─────●──────────────────●════════════════════●──────────────────●─────────────►   │
│       │                  │                    │                  │                  │
│       │              Processing              Done!           Recorded              │
│       │              Data...                                                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Job Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Run to completion** | Pod exits with code 0 = success |
| **Retries on failure** | Configurable backoff limit |
| **Completions tracking** | Counts successful runs |
| **Parallelism** | Run multiple pods simultaneously |
| **Cleanup** | TTL controller or manual deletion |

---

## 🔄 Jobs vs Other Workloads

### Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         WORKLOAD COMPARISON                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Workload      │ Behavior                        │ Use Case                         │
│  ──────────────┼─────────────────────────────────┼──────────────────────────────────│
│  Deployment    │ Run forever, restart on exit    │ Web servers, APIs                │
│  DaemonSet     │ One per node, run forever       │ Logging agents, monitors         │
│  StatefulSet   │ Ordered, stable, run forever    │ Databases                        │
│  Job           │ Run to completion, then stop    │ Batch processing, migrations     │
│  CronJob       │ Scheduled Job                   │ Periodic tasks, backups          │
│                                                                                      │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│                                                                                      │
│  restartPolicy:                                                                     │
│                                                                                      │
│  Deployment/DaemonSet/StatefulSet: Always (required)                               │
│  Job/CronJob: Never or OnFailure (required)                                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Job Types

### Three Patterns

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           JOB PATTERNS                                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  1. SINGLE COMPLETION (Default)                                                     │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  completions: 1 (default)                                                           │
│  parallelism: 1 (default)                                                           │
│                                                                                      │
│  ┌───────┐                                                                          │
│  │ Pod 1 │ ═══════════════════════► Complete!                                       │
│  └───────┘                                                                          │
│                                                                                      │
│  Use: One-time tasks, database migrations                                           │
│                                                                                      │
│  2. FIXED COMPLETION COUNT                                                          │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  completions: 5                                                                     │
│  parallelism: 2                                                                     │
│                                                                                      │
│  ┌───────┐ ┌───────┐                                                               │
│  │ Pod 1 │ │ Pod 2 │ ══════════► (2 complete)                                      │
│  └───────┘ └───────┘                                                               │
│            ┌───────┐ ┌───────┐                                                     │
│            │ Pod 3 │ │ Pod 4 │ ══════════► (4 complete)                            │
│            └───────┘ └───────┘                                                     │
│                      ┌───────┐                                                      │
│                      │ Pod 5 │ ══════════► (5 complete) Job Done!                  │
│                      └───────┘                                                      │
│                                                                                      │
│  Use: Process N items, run N tests                                                 │
│                                                                                      │
│  3. WORK QUEUE (parallelism without fixed completions)                             │
│  ─────────────────────────────────────────────────────────────────────────────────  │
│  completions: unset (or null)                                                       │
│  parallelism: 3                                                                     │
│                                                                                      │
│  ┌───────┐ ┌───────┐ ┌───────┐                                                     │
│  │ Pod 1 │ │ Pod 2 │ │ Pod 3 │ ══► All exit 0 = Complete                           │
│  └───────┘ └───────┘ └───────┘                                                     │
│     ↑         ↑         ↑                                                           │
│     └─────────┼─────────┘                                                           │
│               │                                                                      │
│         Work Queue                                                                  │
│         (Redis, RabbitMQ)                                                           │
│                                                                                      │
│  Use: Queue-based processing, pods coordinate themselves                           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Creating Jobs

### Imperative Creation

```bash
# Create a simple job
kubectl create job hello --image=busybox -- echo "Hello, Kubernetes!"

# Create job from cronjob (run now)
kubectl create job test-job --from=cronjob/my-cronjob

# Generate YAML
kubectl create job hello --image=busybox --dry-run=client -o yaml -- echo "Hello" > job.yaml
```

### Declarative Creation

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.28
        command: ["echo", "Hello, Kubernetes!"]
      restartPolicy: Never    # Required for Jobs
```

```bash
kubectl apply -f job.yaml
```

### Complete Job Example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing-job
spec:
  # Number of successful completions required
  completions: 3
  
  # Number of pods running in parallel
  parallelism: 2
  
  # Deadline in seconds (optional)
  activeDeadlineSeconds: 600    # 10 minutes max
  
  # Retry limit
  backoffLimit: 4               # Retry 4 times on failure
  
  # TTL for cleanup (seconds after completion)
  ttlSecondsAfterFinished: 3600  # Delete after 1 hour
  
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      containers:
      - name: processor
        image: myapp/processor:v1
        command: ["python", "process.py"]
        env:
        - name: BATCH_SIZE
          value: "1000"
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
      
      restartPolicy: OnFailure   # or Never
      
      # Optional: run on specific nodes
      nodeSelector:
        workload: batch
```

---

## 🔢 Job Completions and Parallelism

### Configuration Options

```yaml
spec:
  completions: 5      # Total successful completions needed
  parallelism: 2      # Max concurrent pods
```

### Behavior Matrix

| completions | parallelism | Behavior |
|-------------|-------------|----------|
| 1 (default) | 1 (default) | Single pod runs to completion |
| N | 1 | N pods run sequentially |
| N | M | Up to M pods run at a time until N complete |
| unset | N | Work queue mode - pods coordinate |

### Example: Process 10 Items with 3 Workers

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 10     # Process 10 items
  parallelism: 3      # 3 workers at a time
  template:
    spec:
      containers:
      - name: worker
        image: processor:v1
        command: ["./process-one-item.sh"]
      restartPolicy: Never
```

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         EXECUTION TIMELINE                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Time ──────────────────────────────────────────────────────────────────────────►   │
│                                                                                      │
│  Pod 1: ══════════ (complete)                                                       │
│  Pod 2: ════════════════ (complete)                                                 │
│  Pod 3: ══════ (complete)                                                           │
│                   Pod 4: ══════════ (complete)                                      │
│                   Pod 5: ════════════════ (complete)                                │
│                        Pod 6: ══════ (complete)                                     │
│                                 Pod 7: ══════════ (complete)                        │
│                                      Pod 8: ════════ (complete)                     │
│                                      Pod 9: ══════════════ (complete)               │
│                                           Pod 10: ══════ (complete)                 │
│                                                                                      │
│  At most 3 pods running at any time                                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚠️ Failure Handling

### Restart Policies

```yaml
spec:
  template:
    spec:
      restartPolicy: Never      # Create new pod on failure
      # OR
      restartPolicy: OnFailure  # Restart container in same pod
```

| Policy | Behavior |
|--------|----------|
| `Never` | Failed pod stays failed; Job creates new pod |
| `OnFailure` | Container restarts within same pod |

### Backoff Limit

```yaml
spec:
  backoffLimit: 6    # Default: 6 retries with exponential backoff
```

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         RETRY BACKOFF                                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Retry 1: Wait 10s                                                                  │
│  Retry 2: Wait 20s                                                                  │
│  Retry 3: Wait 40s                                                                  │
│  Retry 4: Wait 80s                                                                  │
│  Retry 5: Wait 160s                                                                 │
│  Retry 6: Wait 320s                                                                 │
│                                                                                      │
│  Max backoff: 6 minutes (capped)                                                    │
│                                                                                      │
│  After backoffLimit reached → Job fails                                             │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Active Deadline

```yaml
spec:
  activeDeadlineSeconds: 600   # Job must complete in 10 minutes
```

- If deadline exceeded, Job terminates all pods
- Job status becomes "Failed"

### Pod Failure Policy (K8s 1.26+)

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob           # Fail entire job
      onExitCodes:
        containerName: main
        operator: In
        values: [42]            # If exit code is 42
    - action: Ignore            # Don't count as failure
      onPodConditions:
      - type: DisruptionTarget  # Node drain, preemption
```

---

## 🧹 Job Cleanup

### TTL After Finished

```yaml
spec:
  ttlSecondsAfterFinished: 3600  # Delete 1 hour after completion
```

```bash
# Completed jobs auto-deleted after TTL
kubectl get jobs   # Job gone after TTL
kubectl get pods   # Pods also cleaned up
```

### Manual Cleanup

```bash
# Delete job and its pods
kubectl delete job hello-job

# Delete all completed jobs
kubectl delete jobs --field-selector status.successful=1

# Delete all failed jobs
kubectl delete jobs --field-selector status.failed=1
```

---

## ⏰ CronJobs

### What is a CronJob?

> **CronJob** creates Jobs on a repeating schedule, similar to cron in Unix/Linux.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           CRONJOB CONCEPT                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  CronJob: "Run this Job every day at 2:00 AM"                                       │
│                                                                                      │
│  schedule: "0 2 * * *"                                                              │
│                                                                                      │
│  Day 1     Day 2     Day 3     Day 4     Day 5                                      │
│  2:00 AM   2:00 AM   2:00 AM   2:00 AM   2:00 AM                                    │
│    │         │         │         │         │                                         │
│    ▼         ▼         ▼         ▼         ▼                                         │
│  ┌───┐     ┌───┐     ┌───┐     ┌───┐     ┌───┐                                      │
│  │Job│     │Job│     │Job│     │Job│     │Job│                                      │
│  └───┘     └───┘     └───┘     └───┘     └───┘                                      │
│    │         │         │         │         │                                         │
│    ▼         ▼         ▼         ▼         ▼                                         │
│  ┌───┐     ┌───┐     ┌───┐     ┌───┐     ┌───┐                                      │
│  │Pod│     │Pod│     │Pod│     │Pod│     │Pod│                                      │
│  └───┘     └───┘     └───┘     └───┘     └───┘                                      │
│    │         │         │         │         │                                         │
│  backup    backup    backup    backup    backup                                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Creating CronJobs

```bash
# Imperative
kubectl create cronjob backup --image=backup:v1 --schedule="0 2 * * *" -- /backup.sh

# Generate YAML
kubectl create cronjob backup --image=backup:v1 --schedule="0 2 * * *" \
  --dry-run=client -o yaml -- /backup.sh > cronjob.yaml
```

### CronJob Manifest

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"           # Every day at 2 AM
  
  # Timezone (K8s 1.27+)
  timeZone: "America/New_York"
  
  # Concurrency policy
  concurrencyPolicy: Forbid        # Don't run if previous still running
  
  # Start deadline
  startingDeadlineSeconds: 200     # Miss if more than 200s late
  
  # Suspend (pause scheduling)
  suspend: false
  
  # History limits
  successfulJobsHistoryLimit: 3    # Keep 3 successful jobs
  failedJobsHistoryLimit: 1        # Keep 1 failed job
  
  # Job template
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:14
            command:
            - /bin/sh
            - -c
            - pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/db.sql
            env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

---

## 📅 Cron Syntax

### Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday = 0)
│ │ │ │ │
* * * * *
```

### Common Examples

| Schedule | Cron Expression | Description |
|----------|-----------------|-------------|
| Every minute | `* * * * *` | Every minute |
| Every hour | `0 * * * *` | At minute 0 of every hour |
| Every day at midnight | `0 0 * * *` | 00:00 daily |
| Every day at 2 AM | `0 2 * * *` | 02:00 daily |
| Every Monday at 9 AM | `0 9 * * 1` | 09:00 on Mondays |
| Every 15 minutes | `*/15 * * * *` | 0, 15, 30, 45 |
| Weekdays at 6 PM | `0 18 * * 1-5` | 18:00 Mon-Fri |
| First of month | `0 0 1 * *` | Midnight on 1st |
| Every 6 hours | `0 */6 * * *` | 00:00, 06:00, 12:00, 18:00 |

### Special Syntax

| Shortcut | Equivalent | Description |
|----------|------------|-------------|
| `@yearly` | `0 0 1 1 *` | Once a year |
| `@monthly` | `0 0 1 * *` | Once a month |
| `@weekly` | `0 0 * * 0` | Once a week |
| `@daily` | `0 0 * * *` | Once a day |
| `@hourly` | `0 * * * *` | Once an hour |

---

## ⚙️ CronJob Configuration

### Concurrency Policy

```yaml
spec:
  concurrencyPolicy: Allow    # Default: allow concurrent jobs
```

| Policy | Behavior |
|--------|----------|
| `Allow` | Multiple jobs can run simultaneously |
| `Forbid` | Skip new job if previous still running |
| `Replace` | Cancel current job, start new one |

### Starting Deadline

```yaml
spec:
  startingDeadlineSeconds: 200
```

- If job misses schedule by more than deadline, it's skipped
- Counts missed schedules when controller was down
- If > 100 schedules missed, CronJob stops creating jobs

### Suspend

```yaml
spec:
  suspend: true   # Pause scheduling
```

```bash
# Suspend a cronjob
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob backup -p '{"spec":{"suspend":false}}'
```

### History Limits

```yaml
spec:
  successfulJobsHistoryLimit: 3  # Default: 3
  failedJobsHistoryLimit: 1      # Default: 1
```

---

## 💼 Common Use Cases

### 1. Database Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 1 * * *"    # Daily at 1 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - mysqldump -h mysql -u root -p$MYSQL_ROOT_PASSWORD --all-databases > /backup/dump-$(date +%Y%m%d).sql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### 2. Log Cleanup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 3 * * 0"    # Weekly on Sunday at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - find /logs -type f -mtime +30 -delete
            volumeMounts:
            - name: logs
              mountPath: /logs
          volumes:
          - name: logs
            persistentVolumeClaim:
              claimName: logs-pvc
          restartPolicy: OnFailure
```

### 3. Data Migration Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:v2
        command: ["./migrate.sh"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
      restartPolicy: Never
```

### 4. Report Generation

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-report
spec:
  schedule: "0 8 * * 1"    # Monday at 8 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: reporter
            image: reporter:v1
            command: ["python", "generate_report.py"]
            env:
            - name: REPORT_TYPE
              value: "weekly"
            - name: EMAIL_TO
              value: "team@company.com"
          restartPolicy: OnFailure
```

---

## 🔧 Operations

### Job Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl create job test --image=busybox -- echo "Hello"
kubectl apply -f job.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get jobs
kubectl get jobs -o wide
kubectl get jobs --show-labels

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe job data-job

# ═══════════════════════════════════════════════════════════════════
# VIEW PODS
# ═══════════════════════════════════════════════════════════════════
kubectl get pods -l job-name=data-job

# ═══════════════════════════════════════════════════════════════════
# LOGS
# ═══════════════════════════════════════════════════════════════════
kubectl logs job/data-job
kubectl logs -l job-name=data-job

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete job data-job
kubectl delete jobs --all
```

### CronJob Commands

```bash
# ═══════════════════════════════════════════════════════════════════
# CREATE
# ═══════════════════════════════════════════════════════════════════
kubectl create cronjob backup --image=backup:v1 --schedule="0 2 * * *" -- /backup.sh
kubectl apply -f cronjob.yaml

# ═══════════════════════════════════════════════════════════════════
# GET / LIST
# ═══════════════════════════════════════════════════════════════════
kubectl get cronjobs
kubectl get cj                    # Short form

# ═══════════════════════════════════════════════════════════════════
# DESCRIBE
# ═══════════════════════════════════════════════════════════════════
kubectl describe cronjob backup

# ═══════════════════════════════════════════════════════════════════
# TRIGGER MANUALLY (create job from cronjob)
# ═══════════════════════════════════════════════════════════════════
kubectl create job manual-backup --from=cronjob/backup

# ═══════════════════════════════════════════════════════════════════
# SUSPEND / RESUME
# ═══════════════════════════════════════════════════════════════════
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'
kubectl patch cronjob backup -p '{"spec":{"suspend":false}}'

# ═══════════════════════════════════════════════════════════════════
# VIEW JOBS CREATED BY CRONJOB
# ═══════════════════════════════════════════════════════════════════
kubectl get jobs -l job-name=backup-xxxxx

# ═══════════════════════════════════════════════════════════════════
# DELETE
# ═══════════════════════════════════════════════════════════════════
kubectl delete cronjob backup
```

---

## 🔧 Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Job never completes | Pod keeps failing | Check logs, fix app |
| Job stuck | Pod pending | Check resources, node capacity |
| CronJob not running | Wrong schedule or suspended | Check schedule syntax |
| Too many jobs | Missing cleanup | Set TTL or history limits |

### Debug Commands

```bash
# Check job status
kubectl describe job my-job

# View pod created by job
kubectl get pods -l job-name=my-job

# Check pod logs
kubectl logs -l job-name=my-job

# Check events
kubectl get events --field-selector involvedObject.name=my-job

# Check cronjob status
kubectl describe cronjob my-cronjob

# See last scheduled time
kubectl get cronjob my-cronjob -o jsonpath='{.status.lastScheduleTime}'
```

---

## 🎓 CKA Exam Tips

### Quick Job Creation

```bash
# Create job with command
kubectl create job hello --image=busybox -- echo "Hello World"

# Create job from cronjob (run immediately)
kubectl create job manual-run --from=cronjob/backup

# Generate YAML
kubectl create job hello --image=busybox --dry-run=client -o yaml -- sh -c "sleep 5; echo done"
```

### Quick CronJob Creation

```bash
# Create cronjob
kubectl create cronjob cleanup --image=busybox --schedule="*/5 * * * *" -- /bin/sh -c "date; echo cleanup done"

# Generate YAML
kubectl create cronjob cleanup --image=busybox --schedule="0 * * * *" --dry-run=client -o yaml -- date
```

### Key Points for Exam

1. **restartPolicy** must be `Never` or `OnFailure` for Jobs
2. **--from=cronjob** to trigger CronJob manually
3. Know cron syntax basics (minute, hour, day, month, weekday)
4. **backoffLimit** controls retry count
5. **ttlSecondsAfterFinished** for auto-cleanup

---

## ✅ Summary

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Job** | Run-to-completion workload |
| **CronJob** | Scheduled Jobs |
| **completions** | Number of successful runs needed |
| **parallelism** | Max concurrent pods |
| **backoffLimit** | Max retries on failure |
| **restartPolicy** | Never or OnFailure (required) |

### Job vs CronJob

| Aspect | Job | CronJob |
|--------|-----|---------|
| Trigger | Manual/one-time | Scheduled |
| Creates | Pods | Jobs (which create Pods) |
| Use case | Batch processing | Periodic tasks |

### Essential Commands

```bash
# Jobs
kubectl create job <name> --image=<image> -- <command>
kubectl get jobs
kubectl logs job/<name>
kubectl delete job <name>

# CronJobs
kubectl create cronjob <name> --image=<image> --schedule="<cron>" -- <command>
kubectl create job <name> --from=cronjob/<cronjob-name>
kubectl get cronjobs
kubectl delete cronjob <name>
```

---

## 🔜 What's Next

In **Chapter 12: Services Deep Dive**, we'll cover:

- Service types (ClusterIP, NodePort, LoadBalancer)
- Service discovery and DNS
- Endpoints and selectors
- Session affinity
- Headless services

---

*Jobs and CronJobs complete the workload story - now you know all workload types!*

