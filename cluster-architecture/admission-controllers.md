# Admission Controllers
Docs: [K8s Docs - Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

## Overview
- Admission controllers intercept requests to the API server **after** authentication and authorization but **before** the object is persisted

### Request Flow
```
Client → API Server → Authentication → Authorization → Admission Controllers → etcd
                                                          ↓
                                                  Mutating → Validating
```

### Types
- **Mutating** admission controllers — can modify the request (e.g., inject default values, add sidecars)
- **Validating** admission controllers — can accept or reject the request (e.g., enforce policies)
- Some controllers are both mutating and validating
- Mutating controllers run first, then validating controllers

## Enabling & Disabling
- Admission controllers are configured on the kube-apiserver via `--enable-admission-plugins` and `--disable-admission-plugins`
- Check current configuration in the API server manifest:

```sh
# view enabled admission plugins
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep enable-admission-plugins

# view disabled admission plugins
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep disable-admission-plugins
```

### Enable an admission controller
Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`:
```yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --enable-admission-plugins=NodeRestriction,ResourceQuota,LimitRanger
```
- The API server restarts automatically after saving (it's a static pod)

## Important Admission Controllers

| Controller | Type | Purpose |
|-----------|------|---------|
| **NamespaceLifecycle** | Validating | Rejects requests in non-existent or terminating namespaces |
| **LimitRanger** | Mutating | Injects default resource requests/limits from a LimitRange |
| **ResourceQuota** | Validating | Enforces resource quotas per namespace |
| **ServiceAccount** | Mutating | Injects the default ServiceAccount and token into pods |
| **NodeRestriction** | Validating | Limits what a kubelet can modify (only its own node and pods) |
| **DefaultStorageClass** | Mutating | Adds default StorageClass to PVCs without one |
| **DefaultTolerationSeconds** | Mutating | Sets default toleration for `notready` and `unreachable` taints |

## Useful Commands
```sh
# check which admission plugins are enabled
kubectl -n kube-system describe pod kube-apiserver-<control-plane> | grep admission

# or check the manifest directly
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission

# check available admission plugins (from the docs or API server help)
kube-apiserver -h | grep enable-admission-plugins
```

## Notes
> 1. Some admission controllers are enabled by default — check the K8s docs for the default list per version.
> 2. Modifying the API server manifest will restart it — the API server may be briefly unavailable.
