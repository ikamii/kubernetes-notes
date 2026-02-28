# Networking General
Docs: [K8s Docs - Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

## Kubernetes Networking Model

Every Kubernetes cluster must satisfy these fundamental requirements:

1. **Pod-to-Pod**: Every pod can communicate with every other pod without NAT
2. **Node-to-Pod**: Agents on a node (e.g., kubelet) can communicate with all pods on that node
3. **Pod IP consistency**: A pod sees its own IP the same way other pods see it

> All of this is implemented by the CNI plugin (see [cni.md](cni.md))

## IP Address Ranges

Kubernetes uses three distinct IP ranges that **must not overlap**:

| Range | Purpose | Configured via |
|-------|---------|----------------|
| **Node network** | IP addresses for nodes themselves | Infrastructure / DHCP |
| **Pod CIDR** | IP addresses for pods | `--cluster-cidr` on controller-manager |
| **Service CIDR** | Virtual IPs for services | `--service-cluster-ip-range` on kube-apiserver |

```sh
# check pod CIDR
kubectl cluster-info dump | grep -m 1 cluster-cidr

# check service CIDR
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range

# check per-node pod CIDR allocation
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

## Pod Networking

### How Pods Get IPs

1. Kubelet invokes the CNI plugin when a pod is scheduled on a node
2. The CNI plugin allocates an IP from the node's pod CIDR range
3. The CNI plugin sets up a **veth pair** — one end in the pod's network namespace, one on the node
4. The pod-side veth becomes `eth0` inside the pod
5. The node-side veth connects to a bridge (e.g., `cni0`, `cbr0`) or overlay network

### Pod-to-Pod Communication

**Same node:**
```
Pod A (eth0) → veth → bridge (cni0) → veth → Pod B (eth0)
```

**Different nodes:**
```
Pod A (eth0) → veth → bridge → node network / overlay → bridge → veth → Pod B (eth0)
```

The overlay mechanism depends on the CNI plugin:
- **VXLAN**: encapsulates L2 frames in UDP (Flannel, Calico)
- **IP-in-IP**: encapsulates IP packets in IP (Calico)
- **BGP**: direct routing without encapsulation (Calico)
- **WireGuard**: encrypted tunnel (Calico, Cilium)

## Linux Networking Fundamentals

Kubernetes networking builds on these Linux primitives:

### Network Namespaces
- Each pod gets its own network namespace — isolated network stack (interfaces, routes, iptables)
- Containers within the same pod share the same network namespace (share `localhost`)

```sh
# list network namespaces on a node
ip netns list

# execute a command in a network namespace
ip netns exec <namespace> ip addr
```

### Virtual Ethernet (veth) Pairs
- A veth pair acts like a virtual network cable connecting two network namespaces
- One end goes in the pod namespace, the other stays in the host namespace

```sh
# see veth interfaces on a node
ip link show type veth
```

### Bridges
- A virtual switch that connects veth pairs on the same node
- CNI plugins create a bridge (e.g., `cni0`) to enable pod-to-pod communication on the same node

```sh
# show bridge interfaces
ip link show type bridge

# show interfaces connected to a bridge
bridge link show
```

## Exploring the Node Network

```sh
# get node IPs
kubectl get nodes -o wide

# get network interfaces on a node
ip a

# find the interface for a specific node IP
ip a | grep <node-ip> -B 2

# show details of a specific interface
ip addr show <interface-name>

# show default gateway
ip route show default

# show all routes
ip route show

# show ARP table
ip neigh show

# check listening ports of control-plane components
netstat -nplt
# or
ss -nplt
```

## Port Forwarding

Forward a local port to a pod port for debugging:

```sh
# forward local port 8080 to pod port 80
kubectl port-forward <pod-name> 8080:80

# forward to a service
kubectl port-forward svc/<service-name> 8080:80

# forward to a deployment
kubectl port-forward deploy/<deployment-name> 8080:80

# listen on all interfaces (not just localhost)
kubectl port-forward --address 0.0.0.0 <pod-name> 8080:80
```

## Network Troubleshooting Tools

```sh
# run a temporary pod with common networking tools
kubectl run netdebug --image=nicolaka/netshoot --rm -it --restart=Never -- /bin/bash

# inside the pod you have access to: curl, wget, nslookup, dig, traceroute, tcpdump, ip, etc.
```

## Useful Commands

```sh
# get nodes with internal/external IPs
kubectl get nodes -o wide

# check which port a service is using
kubectl get svc <service-name> -o jsonpath='{.spec.ports}'

# check the pod network interface from inside
kubectl exec <pod-name> -- ip addr
kubectl exec <pod-name> -- ip route

# check connectivity to another pod
kubectl exec <pod-name> -- wget -qO- -T 5 <target-pod-ip>:<port>

# check DNS resolution
kubectl exec <pod-name> -- nslookup <service-name>

# get all network-related resources
kubectl get svc,endpoints,ingress,networkpolicies -A
```
