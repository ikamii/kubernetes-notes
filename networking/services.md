# Services
Docs: [K8s Docs - Service](https://kubernetes.io/docs/concepts/services-networking/service/)

## Service Types

### ClusterIP (default)
- Exposes the service on an internal cluster IP
- Only reachable from within the cluster
- Use for internal communication between pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### NodePort
- Exposes the service on each node's IP at a static port (range: 30000-32767)
- Accessible from outside the cluster via `<NodeIP>:<NodePort>`
- Automatically creates a ClusterIP to route to

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

### LoadBalancer
- Exposes the service externally using a cloud provider's load balancer
- Automatically creates NodePort and ClusterIP
- Only works with cloud providers that support it

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### ExternalName
- Maps the service to an external DNS name (e.g., `my.database.example.com`)
- No proxying — returns a CNAME record
- No selector, no endpoints

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: my.database.example.com
```

## Headless Services
- Set `clusterIP: None` — no cluster IP is allocated
- DNS returns the individual pod IPs directly instead of a single virtual IP
- Used with StatefulSets for stable network identities
- Each pod gets a DNS record: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

## kube-proxy
Docs: [K8s Docs - Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)

- Runs on every node, maintains network rules for services
- Translates service virtual IPs to pod IPs

### Proxy Modes
- **iptables** (default): uses iptables rules, random pod selection
- **IPVS**: uses IPVS kernel module, supports multiple load balancing algorithms (round-robin, least connections, etc.)

### Check kube-proxy mode
```sh
kubectl logs -n kube-system <kube-proxy-pod> | grep "Using .* Proxier"

# or check the configmap
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode
```

## Endpoints & EndpointSlices
Docs: [K8s Docs - EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

- Endpoints track the IP addresses of pods matching a service's selector
- EndpointSlices are the newer, scalable replacement (split into chunks of 100 endpoints)

### Check endpoints
```sh
kubectl get endpoints <service-name>

# or shorthand
kubectl get ep <service-name>

# check endpointslices
kubectl get endpointslices
```

## SessionAffinity
- `None` (default): random distribution
- `ClientIP`: all requests from the same client IP go to the same pod
```sh
kubectl describe svc <service-name> | grep "Session Affinity"
```

## Get IP Ranges

### IP Range for PODs
```sh
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr
```

### IP Range for Services
```sh
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
```

## Useful Commands
```sh
# create a service imperatively
kubectl expose deployment <deploy-name> --port=80 --target-port=8080 --type=ClusterIP

# create a NodePort service
kubectl expose deployment <deploy-name> --port=80 --type=NodePort

# create a service for a specific pod
kubectl expose pod <pod-name> --port=80 --target-port=8080 --name=<svc-name>

# list all services
kubectl get svc -A

# get service details
kubectl describe svc <service-name>
```
