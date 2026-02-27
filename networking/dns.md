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

## CoreDNS

### Check CoreDNS config file path
```sh
kubectl describe deploy -n kube-system coredns | grep Corefile
```
