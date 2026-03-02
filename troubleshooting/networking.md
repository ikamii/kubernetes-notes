# Debugging Networking
Docs: [K8s Docs - Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/), [K8s Docs - DNS Debugging](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

## DNS Debugging

### Verify DNS is Working

Deploy a pod with DNS tools for testing:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
spec:
  containers:
    - name: dnsutils
      image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
      command: ["sleep", "3600"]
```

```sh
kubectl apply -f dnsutils.yaml

# test DNS resolution
kubectl exec dnsutils -- nslookup kubernetes.default
kubectl exec dnsutils -- nslookup <service-name>.<namespace>.svc.cluster.local
```

### Check resolv.conf

Every pod gets a `/etc/resolv.conf` configured by kubelet:

```sh
kubectl exec <pod-name> -- cat /etc/resolv.conf

# expected output:
# nameserver 10.96.0.10          (CoreDNS ClusterIP)
# search <namespace>.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

- **nameserver** — should point to the CoreDNS service ClusterIP
- **search** — allows short names like `my-svc` to resolve within the namespace
- **ndots:5** — names with fewer than 5 dots get the search domains appended first

### Check CoreDNS

```sh
# check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# check CoreDNS service
kubectl get svc kube-dns -n kube-system

# check CoreDNS endpoints
kubectl get endpointslices -n kube-system -l kubernetes.io/service-name=kube-dns
```

### CoreDNS ConfigMap

```sh
# view CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# edit to enable query logging (add "log" plugin)
kubectl edit configmap coredns -n kube-system
```

Add the `log` plugin inside the Corefile block for debugging:

```
.:53 {
    log        # <-- add this line to enable query logging
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

```sh
# restart CoreDNS to pick up changes
kubectl rollout restart deployment coredns -n kube-system
```

### CoreDNS RBAC

If CoreDNS can't list services/endpoints, it may be a permissions issue:

```sh
# check the CoreDNS ClusterRole
kubectl describe clusterrole system:coredns

# check the ClusterRoleBinding
kubectl describe clusterrolebinding system:coredns
```

### Common DNS Issues

| Issue | Possible Cause | Fix |
|-------|---------------|-----|
| Can't resolve any names | CoreDNS pods not running | Check CoreDNS deployment, restart pods |
| SERVFAIL responses | CoreDNS can't reach upstream DNS | Check `forward` directive in ConfigMap |
| Forwarding loop detected | CoreDNS forwards to itself | Fix `/etc/resolv.conf` on node or use explicit upstream IPs |
| Cross-namespace resolution fails | Wrong DNS name format | Use `<svc>.<namespace>.svc.cluster.local` |
| Slow DNS lookups | `ndots:5` causing extra queries | Set `dnsConfig.options` with lower `ndots` in pod spec |

## Service Connectivity Debugging

### Service Debugging Flow

```
Service not working?
├── Does the Service exist?              → kubectl get svc
├── Does it have Endpoints?              → kubectl get endpoints
│   └── No endpoints?                    → Check selector matches pod labels
├── Is targetPort correct?               → kubectl describe svc
├── Can you reach pod IPs directly?      → kubectl exec -- wget -qO- <pod-ip>:<port>
│   └── Pod not responding?              → Debug the application (see application.md)
├── Can you reach the Service ClusterIP? → kubectl exec -- wget -qO- <cluster-ip>:<port>
│   └── ClusterIP not working?           → Check kube-proxy
└── Does DNS resolve?                    → kubectl exec -- nslookup <svc-name>
    └── DNS not resolving?               → Check CoreDNS (see DNS section above)
```

### Verifying Endpoints

```sh
# check endpoints for a service
kubectl get endpoints <service-name>

# check endpointslices (more detailed)
kubectl get endpointslices -l kubernetes.io/service-name=<service-name>

# describe to see individual endpoint addresses
kubectl describe endpointslice <endpointslice-name>

# if endpoints are empty, check that selectors match pod labels
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
```

### Testing Pod-to-Pod Connectivity

```sh
# get pod IPs
kubectl get pods -o wide

# test connectivity from a temp pod
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- -T 5 <pod-ip>:<port>

# or use curl
kubectl run tmp --image=curlimages/curl --rm -it --restart=Never -- curl -s -m 5 <pod-ip>:<port>
```

### Testing Pod-to-Service

```sh
# test using ClusterIP
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- -T 5 <cluster-ip>:<port>

# test using DNS name
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- -T 5 <service-name>.<namespace>.svc.cluster.local:<port>

# test from within the same namespace (short name)
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- -T 5 <service-name>:<port>
```

### kube-proxy

```sh
# check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# check kube-proxy mode
kubectl logs -n kube-system <kube-proxy-pod> | grep "Using .* Proxier"

# check iptables rules (on a node)
sudo iptables -L -t nat | grep <service-name>

# check IPVS rules (if using IPVS mode)
sudo ipvsadm -L -n
```

## Network Policy Debugging

```sh
# list all network policies across namespaces
kubectl get networkpolicies -A

# describe a specific policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# check which pods a policy selects
kubectl get pods -n <namespace> --show-labels

# test connectivity between pods
kubectl exec <source-pod> -- wget -qO- -T 5 <target-pod-ip>:<port>
```

**Common mistakes with NetworkPolicies:**
- Missing **egress** rules — once you define a policy, all non-matching traffic is denied (both ingress and egress if specified)
- Forgetting to allow **DNS egress** (UDP/TCP port 53) — pods can't resolve service names
- Wrong label selectors in `podSelector` or `namespaceSelector`
- Not realizing policies are **additive** — if any policy allows the traffic, it's allowed

Example: allow DNS egress in a restrictive policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

## CNI Plugin Issues

```sh
# check CNI configuration
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist

# check CNI binaries
ls /opt/cni/bin/

# check kubelet logs for CNI errors
sudo journalctl -u kubelet | grep -i cni

# check if CNI pods are running (e.g., Calico, Flannel, Weave)
kubectl get pods -n kube-system | grep -E "calico|flannel|weave|cilium"
```

**Common CNI issues:**
- Pod stuck in **ContainerCreating** — often a CNI misconfiguration or missing plugin
- No CNI config in `/etc/cni/net.d/` — CNI plugin not installed
- CNI binary missing from `/opt/cni/bin/` — need to install CNI plugins
- IP address exhaustion — pod CIDR range is full

## Useful Commands

```sh
# deploy a debug pod with networking tools
kubectl run netdebug --image=nicolaka/netshoot --rm -it --restart=Never -- /bin/bash

# test DNS resolution
kubectl exec <pod-name> -- nslookup <service-name>

# test HTTP connectivity
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- -T 5 <url>

# check all services and their endpoints
kubectl get svc,endpoints -A

# check kube-proxy config
kubectl get configmap kube-proxy -n kube-system -o yaml

# trace iptables rules for a service
sudo iptables-save | grep <service-name>

# check node networking
ip addr show
ip route show

# check if a port is listening inside a pod
kubectl exec <pod-name> -- netstat -tlnp
kubectl exec <pod-name> -- ss -tlnp
```
