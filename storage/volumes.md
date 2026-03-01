# Volumes & Persistent Storage
Docs: [K8s Docs - Storage](https://kubernetes.io/docs/concepts/storage/)

## Volumes
Docs: [K8s Docs - Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)

- Provide storage to containers in a pod
- Lifetime is tied to the pod (except for persistent volumes)

### emptyDir
- Created when a pod is assigned to a node, deleted when the pod is removed
- Shared between all containers in the same pod
- Use cases: scratch space, caching, sharing files between containers
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: cache
          mountPath: /tmp/cache
  volumes:
    - name: cache
      emptyDir: {}
```

### hostPath
- Mounts a file or directory from the host node's filesystem into a pod
- Data persists beyond the pod's lifetime (but is node-specific)
- Use cases: accessing host logs, Docker socket, node-level config

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: host-logs
          mountPath: /var/log/host
          readOnly: true
  volumes:
    - name: host-logs
      hostPath:
        path: /var/log
        type: Directory          # Directory, DirectoryOrCreate, File, FileOrCreate
```

> **Warning**: `hostPath` is a security risk and should be avoided in production — pods can access the host filesystem.

---

## PersistentVolumes (PV)
Docs: [K8s Docs - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

- Cluster-scoped storage resource provisioned by an admin or dynamically via a StorageClass
- Lifecycle is independent of any pod

### Access Modes
| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Mounted as read-write by a single node |
| ReadOnlyMany | ROX | Mounted as read-only by many nodes |
| ReadWriteMany | RWX | Mounted as read-write by many nodes |
| ReadWriteOncePod | RWOP | Mounted as read-write by a single pod |

### Reclaim Policies
| Policy | Behavior |
|--------|----------|
| **Retain** | PV is kept after PVC is deleted (data preserved, manual cleanup) |
| **Delete** | PV and underlying storage are deleted when PVC is deleted |

### PV Manifest
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:                          # for local/testing — use cloud storage in production
    path: /mnt/data
```

### PV Phases
- `Available` — free, not yet bound to a PVC
- `Bound` — bound to a PVC
- `Released` — PVC deleted, but resource not yet reclaimed
- `Failed` — automatic reclamation failed

---

## PersistentVolumeClaims (PVC)
Docs: [K8s Docs - PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)

- A request for storage by a user
- Kubernetes binds a PVC to a matching PV (by capacity, access mode, storageClass)

### PVC Manifest
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual           # must match PV's storageClassName
```

### Using a PVC in a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-data
```

---

## StorageClasses
Docs: [K8s Docs - Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

- Enable **dynamic provisioning** — PVs are created automatically when a PVC is created
- No need to manually create PVs

### StorageClass Manifest
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"    # make this the default
provisioner: kubernetes.io/aws-ebs       # depends on cloud provider / CSI driver
parameters:
  type: gp3
reclaimPolicy: Delete                     # Delete (default) or Retain
volumeBindingMode: WaitForFirstConsumer   # Immediate or WaitForFirstConsumer
allowVolumeExpansion: true
```

### volumeBindingMode
| Mode | Behavior |
|------|----------|
| `Immediate` | PV is provisioned as soon as the PVC is created |
| `WaitForFirstConsumer` | PV provisioning is delayed until a pod using the PVC is scheduled |

- `WaitForFirstConsumer` is preferred — ensures the PV is created in the same zone as the pod

### Using a StorageClass in a PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast             # references the StorageClass
```
- If no `storageClassName` is specified and a default StorageClass exists, it will be used

---

## Volume Expansion
- Allows growing the size of an existing PVC
- The StorageClass must have `allowVolumeExpansion: true`

### Expand a PVC
```sh
kubectl edit pvc <pvc-name>
# change spec.resources.requests.storage to the new size
```

```yaml
spec:
  resources:
    requests:
      storage: 50Gi    # increased from 20Gi
```

> Some storage backends require the pod to be restarted for the filesystem to be resized.

---

## Useful Commands
```sh
# --- PersistentVolumes ---
kubectl get pv
kubectl describe pv <pv-name>

# --- PersistentVolumeClaims ---
kubectl get pvc
kubectl get pvc -A
kubectl describe pvc <pvc-name>

# --- StorageClasses ---
kubectl get storageclass
kubectl get sc                         # shorthand
kubectl describe sc <name>

# check the default StorageClass
kubectl get sc -o wide

# see which PV is bound to which PVC
kubectl get pv,pvc
```

## Notes
> 1. A PVC can only bind to a PV with **equal or greater** capacity.
> 2. `storageClassName: ""` (empty string) in a PVC explicitly disables dynamic provisioning.
> 3. Deleting a PVC with `reclaimPolicy: Retain` keeps the PV in `Released` state — you must manually clean it up.
