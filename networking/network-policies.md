# Network Policies
Docs: [K8s Docs - Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

## Overview
- Namespace-scoped resources that control traffic flow to/from pods
- Work as a whitelist — once a NetworkPolicy selects a pod, all traffic not explicitly allowed is denied
- Pods without any NetworkPolicy applied accept all traffic by default
- Require a CNI plugin that supports NetworkPolicies (Calico, Weave — Flannel does NOT)

## Key Concepts

### Pod Selector
- `spec.podSelector` selects which pods the policy applies to
- Empty `podSelector: {}` selects all pods in the namespace

### Policy Types
- `Ingress`: controls incoming traffic to selected pods
- `Egress`: controls outgoing traffic from selected pods
- If not specified, defaults to Ingress only (if ingress rules exist)

### Ingress Rules (`from`)
Traffic sources can be defined using:
- **podSelector**: pods in the same namespace matching labels
- **namespaceSelector**: pods in namespaces matching labels
- **ipBlock**: CIDR ranges (e.g., `172.17.0.0/16`), with optional `except`

### Egress Rules (`to`)
Traffic destinations can be defined using:
- **podSelector**: pods in the same namespace matching labels
- **namespaceSelector**: pods in namespaces matching labels
- **ipBlock**: CIDR ranges
- **ports**: specific ports and protocols (TCP/UDP)

### Combining Selectors
- Multiple items in a single `from`/`to` entry = AND (all must match)
- Multiple entries in the `from`/`to` array = OR (any can match)

### Allow ingress from specific pods
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-app
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
```

### Allow ingress from specific namespace
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: production
      ports:
        - protocol: TCP
          port: 80
```

### Allow egress to specific CIDR
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-cidr
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 443
```

## Default Policies

### Default deny all ingress
- Apply a NetworkPolicy with `podSelector: {}` and `policyTypes: ["Ingress"]` with no ingress rules
- Denies all incoming traffic to all pods in the namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Default deny all egress
- Apply a NetworkPolicy with `podSelector: {}` and `policyTypes: ["Egress"]` with no egress rules
- Denies all outgoing traffic from all pods in the namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Default deny all (ingress + egress)
- Combine both policy types with no rules

## Useful Commands
```sh
# list network policies
kubectl get networkpolicies
# or
kubectl get netpol

# describe a network policy
kubectl describe netpol <policy-name> -n <namespace>

# check which pods are selected by a policy
kubectl get pods -n <namespace> -l <label-selector-from-policy>
```

## Notes
1. NetworkPolicies are additive — they never conflict, the union of all policies applies
2. DNS traffic (port 53 TCP/UDP) is often needed in egress rules, otherwise pods can't resolve service names
3. Allow egress to `kube-dns` when applying default deny egress
