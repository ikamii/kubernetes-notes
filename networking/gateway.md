# Gateway API
Docs: [K8s Docs - Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)

## Overview
- The evolution of Ingress — more expressive, extensible, and role-oriented
- Not built into Kubernetes by default — must install the Gateway API CRDs and a compatible controller
- Separates concerns between infrastructure providers (GatewayClass/Gateway) and application developers (Routes)

## Core Resources

### GatewayClass
- Cluster-scoped resource, defines the controller implementation (similar to IngressClass)
- Managed by the infrastructure provider

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-gateway-class
spec:
  controllerName: example.com/gateway-controller
```

```sh
kubectl get gatewayclasses
```

### Gateway
- Namespace-scoped, represents a load balancer instance
- Defines listeners (ports, protocols, hostnames, TLS config)
- References a GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

```sh
kubectl get gateways -A
kubectl describe gateway <gateway-name> -n <namespace>
```

### HTTPRoute
- Namespace-scoped, defines HTTP routing rules
- Attaches to a Gateway via `parentRefs`
- Supports path matching, header matching, query parameter matching
- Supports request/response header modification, URL rewrites, redirects

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /app
      backendRefs:
        - name: app-service
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
```

```sh
kubectl get httproutes -A
kubectl describe httproute <route-name> -n <namespace>
```

### Other Route Types
- **TLSRoute**: TLS passthrough routing (SNI-based)
- **TCPRoute**: TCP traffic routing
- **UDPRoute**: UDP traffic routing
- **GRPCRoute**: gRPC-specific routing

## Gateway vs Ingress
| Feature | Ingress | Gateway API |
|---|---|---|
| Role separation | No | Yes (infra vs app teams) |
| Header matching | No | Yes |
| Traffic splitting | No | Yes (weighted backends) |
| URL rewriting | Controller-specific | Built-in |
| TCP/UDP routing | No | Yes |
| TLS passthrough | Controller-specific | Built-in (TLSRoute) |

## Traffic Splitting
- HTTPRoute supports weighted backends for canary/blue-green deployments
- Assign `weight` to each backend ref to control traffic distribution

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traffic-split
spec:
  parentRefs:
    - name: example-gateway
  hostnames:
    - "example.com"
  rules:
    - backendRefs:
        - name: app-v1
          port: 80
          weight: 80
        - name: app-v2
          port: 80
          weight: 20
```

## TLS
- **Terminate**: TLS terminated at the Gateway, forwards plain HTTP to backends
- **Passthrough**: TLS passed through to the backend pods (via TLSRoute)

## Install Gateway API CRDs
Docs: [Gateway API - Getting Started](https://gateway-api.sigs.k8s.io/guides/)
```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

## Useful Commands
```sh
# check all gateway API resources
kubectl get gatewayclasses,gateways,httproutes -A

# check gateway status and listener conditions
kubectl describe gateway <name> -n <namespace>

# check route status (is it attached to the gateway?)
kubectl describe httproute <name> -n <namespace>
```
