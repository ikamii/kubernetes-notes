# DNS
Docs: [K8s Docs - DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

## DNS Records
### Services
```sh
my-svc.my-namespace.svc.cluster.local
│      │            │   │
│      │            │   └── cluster domain (default: cluster.local)
│      │            └──── service subdomain
│      └──────────────── namespace
└─────────────────────── service name
```

### Pods
#### Pod (direct pod DNS)
```sh
10-244-1-15.default.pod.cluster.local
│           │       │   │
│           │       │   └── cluster domain
│           │       └──── pod subdomain
│           └──────────── namespace
└──────────────────────── pod IP (dashes instead of dots)
```

#### StatefulSet Pod (stable identity)
```sh
web-0.nginx.default.svc.cluster.local
│     │      │       │   │
│     │      │       │   └── cluster domain
│     │      │       └──── service subdomain
│     │      └──────────── namespace
│     └─────────────────── service name (headless)
└───────────────────────── pod hostname
```

## Pod DNS Policy
- `ClusterFirst` (default): queries go to CoreDNS first, then upstream
- `Default`: inherits DNS config from the node
- `None`: allows custom DNS config via `dnsConfig` in the pod spec
- `ClusterFirstWithHostNet`: for pods running with `hostNetwork: true`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  containers:
    - name: nginx
      image: nginx
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 8.8.4.4
    searches:
      - my-namespace.svc.cluster.local
      - svc.cluster.local
    options:
      - name: ndots
        value: "5"
```

## CoreDNS
Docs: [K8s Docs - Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

### Check CoreDNS config file path
```sh
kubectl describe deploy -n kube-system coredns | grep Corefile
```

### CoreDNS Corefile key directives
- `kubernetes`: enables the DNS plugin for the cluster domain
- `forward`: forwards queries not handled by CoreDNS to upstream DNS (e.g., `. /etc/resolv.conf`)
- `cache`: caches DNS responses
- `loop`: detects and stops forwarding loops
- `errors`: logs errors

### Edit CoreDNS ConfigMap
```sh
kubectl edit cm coredns -n kube-system
```

### Troubleshooting DNS
```sh
# run a test pod and perform DNS lookup
kubectl run dns-test --image=busybox:1.28 --restart=Never --rm -it -- nslookup <service-name>

# lookup from an existing pod
kubectl exec -it <pod-name> -- nslookup <service-name>.<namespace>.svc.cluster.local

# check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```
