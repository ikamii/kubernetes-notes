# Workload Resources
Docs: [K8s Docs - Workloads](https://kubernetes.io/docs/concepts/workloads/)

## Pods
Docs: [K8s Docs - Pods](https://kubernetes.io/docs/concepts/workloads/pods/)

- Smallest deployable unit in Kubernetes
- A pod runs one or more containers that share the same network namespace and storage

### Pod spec essentials
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  restartPolicy: Always          # Always (default), OnFailure, Never
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

### Multi-container patterns

**Init Containers** — run to completion before app containers start
```yaml
spec:
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nslookup db-service; do sleep 2; done']
  containers:
    - name: app
      image: myapp:1.0
```

**Sidecar Containers** — run alongside the main container for the pod's lifetime
- Common use cases: log collectors, proxies, config reloaders
```yaml
spec:
  initContainers:
    - name: log-shipper
      image: fluentd
      restartPolicy: Always       # makes it a sidecar (runs for pod's lifetime)
  containers:
    - name: app
      image: myapp:1.0
```

### Restart Policies
- `Always` (default) — always restart, used by Deployments/StatefulSets/DaemonSets
- `OnFailure` — restart only on non-zero exit code, used by Jobs
- `Never` — never restart

---

## Probes
Docs: [K8s Docs - Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

- Probes let the kubelet check container health and control traffic routing
- Three probe types serve different purposes:

| Probe | Purpose | Failure action |
|-------|---------|---------------|
| **Liveness** | Is the container still running? | Restarts the container |
| **Readiness** | Is the container ready to serve traffic? | Removes pod from Service endpoints |
| **Startup** | Has the container finished starting? | Blocks liveness/readiness until it succeeds |

### Probe mechanisms
- **httpGet** — HTTP GET to a path/port; 200–399 = success
- **tcpSocket** — TCP connection to a port; connection established = success
- **exec** — runs a command inside the container; exit code 0 = success

### Common fields
```yaml
initialDelaySeconds: 5      # wait before first probe (default 0)
periodSeconds: 10            # interval between probes (default 10)
timeoutSeconds: 1            # probe timeout (default 1)
failureThreshold: 3          # failures before taking action (default 3)
successThreshold: 1          # successes to be considered healthy (default 1)
```

### Liveness probe — httpGet
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
```

### Readiness probe — tcpSocket
```yaml
spec:
  containers:
    - name: db
      image: postgres:16
      readinessProbe:
        tcpSocket:
          port: 5432
        initialDelaySeconds: 5
        periodSeconds: 10
```

### Startup probe — exec
```yaml
spec:
  containers:
    - name: legacy-app
      image: legacy:2.0
      startupProbe:
        exec:
          command: ["cat", "/tmp/started"]
        failureThreshold: 30
        periodSeconds: 10          # gives up to 30×10 = 300s to start
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 5
```

- Use startup probes for slow-starting containers to avoid premature liveness kills
- Liveness and readiness probes do not run until the startup probe succeeds

---

## ReplicaSets
Docs: [K8s Docs - ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

- Ensures a specified number of pod replicas are running at all times
- Uses label selectors to identify which pods it manages
- Rarely created directly — Deployments manage ReplicaSets automatically

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

---

## Deployments
Docs: [K8s Docs - Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

- Manages ReplicaSets and provides declarative updates for pods
- Supports rolling updates and rollbacks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate            # RollingUpdate (default) or Recreate
    rollingUpdate:
      maxSurge: 1                  # max pods above desired count during update
      maxUnavailable: 0            # max pods unavailable during update
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

### Strategy types
- **RollingUpdate** (default) — gradually replaces old pods with new ones
  - `maxSurge` — how many extra pods can be created (absolute number or percentage)
  - `maxUnavailable` — how many pods can be unavailable (absolute number or percentage)
- **Recreate** — kills all existing pods before creating new ones (causes downtime)

### Rolling updates & rollbacks
```sh
# update the image (triggers a rolling update)
kubectl set image deployment/nginx-deploy nginx=nginx:1.26

# check rollout status
kubectl rollout status deployment/nginx-deploy

# view rollout history
kubectl rollout history deployment/nginx-deploy

# view a specific revision
kubectl rollout history deployment/nginx-deploy --revision=2

# rollback to the previous revision
kubectl rollout undo deployment/nginx-deploy

# rollback to a specific revision
kubectl rollout undo deployment/nginx-deploy --to-revision=1

# pause and resume a rollout
kubectl rollout pause deployment/nginx-deploy
kubectl rollout resume deployment/nginx-deploy
```

---

## StatefulSets
Docs: [K8s Docs - StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

- For stateful applications that require stable network identity and persistent storage
- Pods are created in order (0, 1, 2...) and deleted in reverse order
- Each pod gets a stable hostname: `<statefulset-name>-<ordinal>`
- Requires a **headless service** (`clusterIP: None`)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless     # must reference a headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:               # each pod gets its own PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

- DNS for each pod: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`
- e.g., `postgres-0.postgres-headless.default.svc.cluster.local`

---

## DaemonSets
Docs: [K8s Docs - DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

- Ensures a copy of a pod runs on **every** node (or a subset of nodes)
- When a new node joins the cluster, the DaemonSet pod is automatically added
- Use cases: log collectors, monitoring agents, network plugins, storage daemons

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule        # allow scheduling on control plane nodes
      containers:
        - name: fluentd
          image: fluentd:v1.16
```

---

## Jobs
Docs: [K8s Docs - Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

- Runs pods to completion — creates one or more pods and ensures they succeed
- Pods are not restarted after successful completion

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calc
spec:
  completions: 5                    # total successful completions needed
  parallelism: 2                    # how many pods run at the same time
  backoffLimit: 4                   # max retries before marking as failed
  activeDeadlineSeconds: 300        # timeout for the entire job
  template:
    spec:
      restartPolicy: OnFailure      # must be OnFailure or Never
      containers:
        - name: pi
          image: perl:5.34
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```

---

## CronJobs
Docs: [K8s Docs - CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

- Creates Jobs on a repeating schedule (cron syntax)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"             # minute hour day-of-month month day-of-week
  concurrencyPolicy: Forbid          # Allow (default), Forbid, Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200       # deadline if a scheduled run is missed
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox
              command: ["/bin/sh", "-c", "echo Running backup at $(date)"]
```

### Schedule syntax
```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sun=0)
│ │ │ │ │
* * * * *
```

### concurrencyPolicy
- `Allow` (default) — allows concurrent Job runs
- `Forbid` — skips the new run if the previous is still running
- `Replace` — cancels the currently running Job and starts a new one

---

## Useful Commands
```sh
# --- Pods ---
kubectl run nginx --image=nginx                           # create a pod
kubectl run nginx --image=nginx --dry-run=client -o yaml  # generate YAML
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name> -c <container-name>               # logs for a specific container
kubectl exec -it <pod-name> -- /bin/sh

# --- Deployments ---
kubectl create deployment nginx --image=nginx --replicas=3
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.26
kubectl rollout status deployment/nginx
kubectl rollout undo deployment/nginx

# --- StatefulSets ---
kubectl get statefulsets
kubectl describe statefulset <name>
kubectl scale statefulset <name> --replicas=5

# --- DaemonSets ---
kubectl get daemonsets -A
kubectl describe daemonset <name>

# --- Jobs & CronJobs ---
kubectl create job test-job --image=busybox -- echo "hello"
kubectl get jobs
kubectl get cronjobs
kubectl create cronjob test-cron --image=busybox --schedule="*/5 * * * *" -- echo "hello"
kubectl delete job <job-name>
```
