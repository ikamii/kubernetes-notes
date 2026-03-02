# Monitoring
Docs: [K8s Docs - Metrics Server](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)

## metrics-server

### Overview
- Cluster-wide aggregator of resource usage data (CPU and memory)
- Collects metrics from kubelets on each node
- Required for `kubectl top` and Horizontal Pod Autoscaler (HPA) to work
- Does **not** store historical data — provides current point-in-time metrics only

### Installing metrics-server
```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> In test/lab environments, you may need to add `--kubelet-insecure-tls` to the metrics-server deployment args.

### Verifying metrics-server
```sh
# check if metrics-server pod is running
kubectl get pods -n kube-system | grep metrics-server

# check if the API is available
kubectl get apiservices | grep metrics

# should show v1beta1.metrics.k8s.io with AVAILABLE=True
```

## kubectl top

### Node metrics
```sh
# show CPU and memory usage for all nodes
kubectl top nodes

# sort by CPU usage
kubectl top nodes --sort-by=cpu

# sort by memory usage
kubectl top nodes --sort-by=memory
```

### Pod metrics
```sh
# show CPU and memory usage for pods in current namespace
kubectl top pods

# show metrics for all namespaces
kubectl top pods -A

# show metrics for a specific pod
kubectl top pod <pod-name>

# show container-level metrics
kubectl top pod <pod-name> --containers

# sort by CPU usage
kubectl top pods --sort-by=cpu

# sort by memory usage
kubectl top pods --sort-by=memory

# show metrics for pods with a specific label
kubectl top pods -l app=nginx
```

## Useful Commands
```sh
# quick check — is metrics-server working?
kubectl top nodes
kubectl top pods -A

# check metrics-server logs if not working
kubectl logs -n kube-system deployment/metrics-server
```
