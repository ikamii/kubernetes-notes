# Ingress
Docs: [K8s Docs - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## Ingress Controllers
Docs: [K8s Docs - Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

- An Ingress resource does nothing without an Ingress Controller deployed
- The controller watches for Ingress resources and configures the underlying load balancer/proxy
- Common controllers: NGINX Ingress Controller (most common in CKA), HAProxy, Traefik

## IngressClass
- Defines which controller should handle an Ingress resource
- Multiple controllers can coexist in a cluster, each with its own IngressClass
- Set a default IngressClass with the annotation `ingressclass.kubernetes.io/is-default-class: "true"`

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

```sh
# list available ingress classes
kubectl get ingressclass
```

## Routing Rules

### Path-based routing
- Routes traffic to different backends based on the URL path
- Path types:
  - `Prefix`: matches the URL path prefix (e.g., `/app` matches `/app`, `/app/foo`)
  - `Exact`: matches the exact URL path only (e.g., `/app` matches only `/app`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
          - path: /api
            pathType: Exact
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

### Host-based routing
- Routes traffic to different backends based on the `Host` header
- Each host can have its own set of path rules

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

## Default Backend
- Handles requests that don't match any rule (host or path)
- Returns 404 if no default backend is configured
- Can be set as `defaultBackend` in the Ingress spec or configured on the controller

## TLS Termination
- Ingress can terminate TLS using a Kubernetes Secret of type `kubernetes.io/tls`
- The Secret must contain `tls.crt` and `tls.key`
- TLS is configured per-host in the Ingress `tls` section

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

### Create TLS Secret
```sh
kubectl create secret tls <secret-name> --cert=path/to/cert --key=path/to/key
```

## Useful Commands
```sh
# check host and rules
kubectl describe ingress -n <namespace> <ingress-resource-name>

# Check under Rules:
Rules:
  Host        Path  Backends
  ----        ----  --------

# create ingress imperatively
kubectl create ingress <name> --rule="host/path=service:port"

# create ingress with TLS
kubectl create ingress <name> --rule="host/path=service:port,tls=secret-name"

# list all ingress resources
kubectl get ingress -A
```
