# etcd Backup & Restore
Docs: [K8s Docs - Operating etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

## Overview
- etcd is the key-value store that holds all cluster data (pods, services, secrets, configmaps, etc.)
- Runs as a static pod on control plane nodes — managed by kubelet, not the API server
- Manifest location: `/etc/kubernetes/manifests/etcd.yaml`
- Data directory: `/var/lib/etcd` (default)
- Losing etcd = losing the entire cluster state

## etcd Configuration
- All connection details can be found in the etcd static pod manifest

### Find etcd endpoints, certs, and keys
```sh
# check the etcd pod manifest
cat /etc/kubernetes/manifests/etcd.yaml
```

Key flags to look for:
- `--listen-client-urls` — etcd endpoint (e.g., `https://127.0.0.1:2379`)
- `--cert-file` — etcd server certificate
- `--key-file` — etcd server key
- `--trusted-ca-file` — CA certificate

### Check etcd health
```sh
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

## Backup

### Take a snapshot
```sh
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

### Verify the snapshot
```sh
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-table
```

## Restore

### 1. Restore the snapshot to a new data directory
```sh
ETCDCTL_API=3 etcdctl \
  --data-dir=/var/lib/etcd-restored \
  snapshot restore /opt/etcd-backup.db
```

### 2. Update the etcd static pod manifest
Edit `/etc/kubernetes/manifests/etcd.yaml` to point to the new data directory:
- Change `--data-dir=/var/lib/etcd` to `--data-dir=/var/lib/etcd-restored`
- Update the volume `hostPath` from `/var/lib/etcd` to `/var/lib/etcd-restored`

```yaml
# in the etcd pod spec
volumes:
  - name: etcd-data
    hostPath:
      path: /var/lib/etcd-restored   # updated path
      type: DirectoryOrCreate
```

### 3. Wait for etcd to restart
- kubelet watches `/etc/kubernetes/manifests/` and will automatically restart the etcd pod
- It may take a minute for the API server to come back

### 4. Verify the restore
```sh
kubectl get pods -A
```

## Useful Commands
```sh
# check etcd pod is running
kubectl get pods -n kube-system | grep etcd

# check etcd version
kubectl describe pod etcd-<control-plane> -n kube-system | grep Image

# list etcd members
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# check if etcdctl is installed
etcdctl version
```

## Notes
> 1. Always use `ETCDCTL_API=3` — version 2 API is deprecated.
> 2. The cert paths may vary depending on your cluster setup — always check the etcd pod manifest first.
> 3. When restoring, use a new `--data-dir` to avoid corrupting the existing data.
