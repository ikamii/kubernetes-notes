# Scheduling
Docs: [K8s Docs - Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)

## Resource Requests & Limits
Docs: [K8s Docs - Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

### CPU & Memory Units
- **CPU**: `1` = 1 vCPU/core, `100m` = 0.1 CPU (millicores)
- **Memory**: `128Mi` (mebibytes), `1Gi` (gibibytes), `256M` (megabytes)

### Requests vs Limits
- **Requests** — minimum resources guaranteed to the container; used by the scheduler for placement
- **Limits** — maximum resources the container can use; exceeding memory limit = OOMKilled, exceeding CPU = throttled

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### QoS Classes
| Class | Condition |
|-------|-----------|
| **Guaranteed** | Every container has equal requests and limits for both CPU and memory |
| **Burstable** | At least one container has a request or limit set (but not Guaranteed) |
| **BestEffort** | No requests or limits set on any container |

- Under memory pressure, BestEffort pods are evicted first, then Burstable, then Guaranteed

### LimitRange
- Sets default/min/max resource constraints per container in a namespace
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:          # default limits
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:   # default requests
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

### ResourceQuota
- Limits total resource consumption per namespace
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
```

---

## nodeSelector
- Simplest way to constrain pods to nodes with specific labels
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

```sh
# label a node
kubectl label nodes <node-name> disktype=ssd
```

---

## Node Affinity
Docs: [K8s Docs - Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)

- More expressive than nodeSelector — supports `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`

### requiredDuringSchedulingIgnoredDuringExecution (hard requirement)
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
```

### preferredDuringSchedulingIgnoredDuringExecution (soft preference)
```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

---

## Pod Affinity & Anti-Affinity
Docs: [K8s Docs - Pod Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

- Schedule pods relative to other pods (co-locate or spread)
- `topologyKey` defines the domain (e.g., `kubernetes.io/hostname` for per-node, `topology.kubernetes.io/zone` for per-zone)

### Pod Affinity — co-locate with matching pods
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname
```

### Pod Anti-Affinity — spread away from matching pods
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
```

---

## Taints & Tolerations
Docs: [K8s Docs - Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

- **Taints** are applied to nodes — repel pods that don't tolerate them
- **Tolerations** are applied to pods — allow scheduling on tainted nodes

### Taint effects
| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New pods won't be scheduled (existing pods stay) |
| `PreferNoSchedule` | Scheduler tries to avoid the node (soft) |
| `NoExecute` | New pods won't be scheduled AND existing non-tolerating pods are evicted |

### Managing taints
```sh
# add a taint
kubectl taint nodes <node-name> key=value:NoSchedule

# remove a taint (note the trailing -)
kubectl taint nodes <node-name> key=value:NoSchedule-

# view taints on a node
kubectl describe node <node-name> | grep Taints
```

### Tolerating a taint
```yaml
spec:
  tolerations:
    - key: "key"
      operator: "Equal"        # Equal or Exists
      value: "value"
      effect: "NoSchedule"
```

### Tolerate NoExecute with eviction delay
```yaml
spec:
  tolerations:
    - key: "node.kubernetes.io/unreachable"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300     # pod stays for 300s after taint is applied
```

---

## Pod Topology Spread Constraints
Docs: [K8s Docs - Topology Spread](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

- Distribute pods evenly across failure domains (zones, nodes)
```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                              # max difference between domains
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule        # DoNotSchedule or ScheduleAnyway
      labelSelector:
        matchLabels:
          app: web
```

---

## Pod Disruption Budgets
Docs: [K8s Docs - PDB](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)

- Limits the number of pods that can be simultaneously down during voluntary disruptions (drain, upgrade)
- Use `minAvailable` OR `maxUnavailable` (not both)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2               # at least 2 pods must always be running
  # or: maxUnavailable: 1       # at most 1 pod can be unavailable
  selector:
    matchLabels:
      app: web
```

---

## Horizontal Pod Autoscaler (HPA)
Docs: [K8s Docs - HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

- Automatically scales the number of pod replicas based on observed metrics
- Requires **metrics-server** to be installed

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Imperative creation
```sh
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70
```

---

## Useful Commands
```sh
# --- Resources ---
kubectl describe node <node-name> | grep -A5 "Allocated resources"
kubectl get limitrange -n <namespace>
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <name> -n <namespace>

# --- Labels & Scheduling ---
kubectl label nodes <node-name> key=value
kubectl label nodes <node-name> key-                         # remove label
kubectl get nodes --show-labels
kubectl taint nodes <node-name> key=value:NoSchedule
kubectl taint nodes <node-name> key=value:NoSchedule-        # remove taint

# --- PDB ---
kubectl get pdb
kubectl describe pdb <name>

# --- HPA ---
kubectl get hpa
kubectl describe hpa <name>
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=70
```
