# Debugging Clusters
Docs: [K8s Docs - Debug Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

## Cluster Health Check

```sh
# cluster info and component endpoints
kubectl cluster-info

# list all nodes and their status
kubectl get nodes -o wide

# component status (deprecated but still seen in CKA)
kubectl get componentstatuses

# dump cluster state for detailed debugging
kubectl cluster-info dump --output-directory=/tmp/cluster-dump
```

## Node Troubleshooting

### Node Conditions

| Condition | Description |
|-----------|-------------|
| **Ready** | `True` if the node is healthy and ready to accept pods |
| **NotReady** | Node is not healthy — kubelet not running, network issues, etc. |
| **MemoryPressure** | Node is running low on memory |
| **DiskPressure** | Node is running low on disk space |
| **PIDPressure** | Too many processes running on the node |
| **NetworkUnavailable** | Network is not configured correctly on the node |

```sh
# check node conditions
kubectl describe node <node-name> | grep -A 10 "Conditions"
```

### Debugging a NotReady Node

Step-by-step approach:

```sh
# 1. identify NotReady nodes
kubectl get nodes

# 2. describe the node — check Conditions, Taints, Events
kubectl describe node <node-name>

# 3. SSH to the node and check kubelet
ssh <node-name>
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50 --no-pager

# 4. check container runtime
sudo systemctl status containerd
sudo crictl ps

# 5. check kubelet certificates
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -noout -dates

# 6. check kubelet config
cat /var/lib/kubelet/config.yaml
```

### Node Resource Issues

```sh
# check resource usage across nodes
kubectl top nodes

# check allocated vs available resources on a node
kubectl describe node <node-name>
# look at "Allocated resources" section — shows requests and limits

# find pods consuming the most resources
kubectl top pods -A --sort-by=memory
kubectl top pods -A --sort-by=cpu
```

## Control Plane Troubleshooting

### Static Pod Manifests

Control plane components run as static pods managed by kubelet. Manifests are located at:

```
/etc/kubernetes/manifests/
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
├── kube-scheduler.yaml
└── etcd.yaml
```

```sh
# check static pod manifests
ls /etc/kubernetes/manifests/

# view a specific manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# after editing a manifest, kubelet auto-restarts the component
# monitor with:
crictl ps | grep kube-apiserver
```

### Control Plane Logs

| Component | Log Location | journalctl |
|-----------|-------------|------------|
| kube-apiserver | `/var/log/kube-apiserver.log` | `crictl logs <container-id>` |
| kube-scheduler | `/var/log/kube-scheduler.log` | `crictl logs <container-id>` |
| kube-controller-manager | `/var/log/kube-controller-manager.log` | `crictl logs <container-id>` |
| etcd | `/var/log/etcd.log` | `crictl logs <container-id>` |
| kubelet | — | `journalctl -u kubelet` |

```sh
# find control plane container IDs
crictl ps | grep kube-apiserver
crictl ps | grep kube-scheduler
crictl ps | grep kube-controller-manager
crictl ps | grep etcd

# view logs for a control plane component
crictl logs <container-id>

# kubelet logs (runs as a systemd service, not a container)
sudo journalctl -u kubelet -n 100 --no-pager
sudo journalctl -u kubelet -f    # follow live

# check control plane pods via kubectl
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-apiserver-<node-name>
```

### Common Control Plane Issues

**API server not starting:**
- Check the static pod manifest for syntax errors: `cat /etc/kubernetes/manifests/kube-apiserver.yaml`
- Check certificates: `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates`
- Check etcd connectivity: `crictl logs <etcd-container-id>`
- Check kubelet logs: `journalctl -u kubelet | grep apiserver`

**Scheduler not scheduling pods:**
- Check scheduler logs: `kubectl logs -n kube-system kube-scheduler-<node-name>`
- Check leader election: `kubectl get endpoints kube-scheduler -n kube-system -o yaml`
- Verify scheduler is running: `kubectl get pods -n kube-system | grep scheduler`

**Controller manager issues:**
- Check logs: `kubectl logs -n kube-system kube-controller-manager-<node-name>`
- Check RBAC permissions
- Verify leader election: `kubectl get endpoints kube-controller-manager -n kube-system -o yaml`

## Cluster Failure Modes

| Failure | Impact |
|---------|--------|
| **API server down** | Cannot create/update/delete resources; existing pods continue running |
| **etcd data loss** | Cluster state lost; requires backup restore (`etcdctl snapshot restore`) |
| **Node disconnected** | Node marked **NotReady** after ~40s; pods evicted after ~5min |
| **Network partition** | Split-brain scenarios; nodes may disagree on cluster state |
| **Scheduler down** | New pods stay **Pending**; existing pods unaffected |
| **Controller manager down** | No reconciliation (replicas not maintained, no garbage collection) |

## Worker Node Operations

```sh
# mark node as unschedulable (no new pods, existing pods stay)
kubectl cordon <node-name>

# mark node as schedulable again
kubectl uncordon <node-name>

# drain node — evict all pods and mark unschedulable
kubectl drain <node-name> --ignore-daemonsets

# drain with additional flags
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force                        # force eviction of standalone pods

# check taints on a node
kubectl describe node <node-name> | grep -i taint

# add a taint
kubectl taint nodes <node-name> key=value:NoSchedule

# remove a taint
kubectl taint nodes <node-name> key=value:NoSchedule-
```

## Useful Commands

```sh
# quick cluster health check
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info

# check all events cluster-wide
kubectl get events -A --sort-by='.lastTimestamp'

# check node conditions
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason

# find pods on a specific node
kubectl get pods -A --field-selector spec.nodeName=<node-name>

# check kubelet status on a node
ssh <node-name> "sudo systemctl status kubelet"

# restart kubelet
ssh <node-name> "sudo systemctl restart kubelet"

# check certificates expiration
kubeadm certs check-expiration

# renew certificates
kubeadm certs renew all
```
