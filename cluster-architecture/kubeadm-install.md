# Kubeadm Cluster Installation
Docs: [K8s Docs - Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

## Prerequisites

### Disable swap
```sh
sudo swapoff -a
# make it permanent — comment out swap entry in /etc/fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Load required kernel modules
```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Set sysctl parameters
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### Required ports
| Component | Port | Protocol |
|-----------|------|----------|
| kube-apiserver | 6443 | TCP |
| etcd | 2379-2380 | TCP |
| kubelet | 10250 | TCP |
| kube-scheduler | 10259 | TCP |
| kube-controller-manager | 10257 | TCP |
| NodePort Services | 30000-32767 | TCP |

### Install container runtime (containerd)
```sh
sudo apt-get update
sudo apt-get install -y containerd

# configure containerd to use systemd cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# set SystemdCgroup = true in the config
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
```

## Installing kubeadm, kubelet, kubectl
```sh
# add the Kubernetes apt repository
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# prevent automatic upgrades
sudo apt-mark hold kubelet kubeadm kubectl
```

## Initializing Control Plane
```sh
# initialize with a pod network CIDR (required for most CNI plugins)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# set up kubeconfig for the current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Joining Worker Nodes
```sh
# the join command is printed after kubeadm init — run this on each worker node
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Token management
```sh
# list existing tokens
kubeadm token list

# create a new token (if the original expired — tokens expire after 24h)
kubeadm token create --print-join-command
```

## Installing a CNI Plugin
- Pods will remain in `Pending` state until a CNI plugin is installed
- Example with Flannel:
```sh
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

- Example with Calico:
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Useful Commands
```sh
# check cluster status after init
kubectl get nodes
kubectl get pods -n kube-system

# check kubelet status
systemctl status kubelet
journalctl -u kubelet -f

# check if all system pods are running
kubectl get pods -n kube-system -o wide

# reset a node (removes all kubeadm state — use with caution)
sudo kubeadm reset
```

## Notes
> 1. The `--pod-network-cidr` must match what the CNI plugin expects (e.g., `10.244.0.0/16` for Flannel).
> 2. Always install the CNI plugin before joining worker nodes.
> 3. Join tokens expire after 24 hours — use `kubeadm token create --print-join-command` to generate a new one.
