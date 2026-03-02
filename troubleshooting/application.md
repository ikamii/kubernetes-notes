# Debugging Applications
Docs: [K8s Docs - Debug Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)

## Pod Lifecycle & Status

### Pod Phases

| Phase | Description |
|-------|-------------|
| **Pending** | Pod accepted but not yet scheduled or images not pulled |
| **Running** | Pod bound to a node, all containers created, at least one running |
| **Succeeded** | All containers terminated successfully, won't restart |
| **Failed** | All containers terminated, at least one failed |
| **Unknown** | Pod state cannot be determined (usually node communication failure) |

### Container States

| State | Description |
|-------|-------------|
| **Waiting** | Container not yet running (pulling image, applying secrets, etc.) |
| **Running** | Container executing without issues |
| **Terminated** | Container ran to completion or failed |

```sh
# check pod phase
kubectl get pod <pod-name> -o jsonpath='{.status.phase}'

# check container states
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state}'
```

## Debugging Workflow

Step-by-step approach for debugging any pod issue:

```sh
# 1. get pod status
kubectl get pod <pod-name> -o wide

# 2. describe pod — check Events, Conditions, container states
kubectl describe pod <pod-name>

# 3. check logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>    # specific container
kubectl logs <pod-name> --previous              # previous crashed container

# 4. exec into the container
kubectl exec -it <pod-name> -- /bin/sh

# 5. debug with ephemeral container (when exec is not available)
kubectl debug <pod-name> -it --image=busybox
```

## Common Pod Issues

### Pending

Pod stays in **Pending** state — the scheduler cannot place the pod on a node.

**Common causes:**
- Insufficient CPU/memory on nodes
- Node selectors or affinity rules don't match any node
- Taints on nodes with no matching tolerations
- PersistentVolumeClaim not bound

```sh
# check why pod is pending
kubectl describe pod <pod-name>
# look at Events section for scheduling errors

# check cluster events
kubectl get events --sort-by='.lastTimestamp'

# check node resources
kubectl top nodes
kubectl describe node <node-name>
# look at "Allocated resources" section
```

### ImagePullBackOff / ErrImagePull

Pod cannot pull the container image.

**Common causes:**
- Wrong image name or tag
- Image doesn't exist in the registry
- Missing or incorrect registry credentials (ImagePullSecrets)
- Private registry with no auth configured

```sh
# check the Events section for the exact error
kubectl describe pod <pod-name>

# verify the image name in the pod spec
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'

# check if imagePullSecrets are configured
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'
```

### CrashLoopBackOff

Container starts, crashes, restarts, and keeps crashing in a loop.

**Common causes:**
- Application error or exception on startup
- Liveness probe failing
- Missing configuration (env vars, config files, secrets)
- Wrong command or entrypoint
- Insufficient resources (OOMKilled)

```sh
# check logs from the current (crashing) container
kubectl logs <pod-name>

# check logs from the previous crashed container
kubectl logs <pod-name> --previous

# check exit code and reason
kubectl describe pod <pod-name>
# look at "Last State" — exit code 137 = OOMKilled, 1 = app error

# check if it's OOMKilled
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

### CreateContainerConfigError

Container cannot start because a referenced ConfigMap or Secret is missing.

**Common causes:**
- ConfigMap or Secret referenced in `envFrom` or `volumes` doesn't exist
- Key referenced in `valueFrom.configMapKeyRef` or `secretKeyRef` doesn't exist

```sh
# check Events for the exact missing resource
kubectl describe pod <pod-name>

# verify ConfigMaps exist
kubectl get configmap -n <namespace>

# verify Secrets exist
kubectl get secret -n <namespace>
```

### Pod Running but Misbehaving

Pod is in **Running** state but the application doesn't behave as expected.

**Common causes:**
- Wrong command or args in the pod spec
- Incorrect environment variables
- YAML typos (indentation, field names)
- Wrong port configuration

```sh
# compare running spec with expected spec
kubectl get pod <pod-name> -o yaml

# check environment variables inside the container
kubectl exec <pod-name> -- env

# check logs for application errors
kubectl logs <pod-name> -f

# exec into the container to investigate
kubectl exec -it <pod-name> -- /bin/sh
```

### Terminating (Stuck)

Pod is stuck in **Terminating** state and won't delete.

**Common causes:**
- Finalizers preventing deletion
- Admission webhooks blocking deletion
- Node is unreachable

```sh
# check for finalizers
kubectl get pod <pod-name> -o jsonpath='{.metadata.finalizers}'

# check pod details
kubectl describe pod <pod-name>

# force delete (use with caution)
kubectl delete pod <pod-name> --force --grace-period=0
```

## Debugging with kubectl debug

### Ephemeral Containers

Attach a debug container to a running pod (useful when the pod image has no shell or tools):

```sh
# attach a busybox debug container
kubectl debug <pod-name> -it --image=busybox

# share process namespace to see processes in other containers
kubectl debug <pod-name> -it --image=busybox --share-processes

# target a specific container (share its PID namespace)
kubectl debug <pod-name> -it --image=busybox --target=<container-name>
```

### Copy-based Debugging

Create a copy of the pod with modifications for debugging:

```sh
# copy pod with a different command
kubectl debug <pod-name> -it --copy-to=debug-pod --container=<container-name> -- /bin/sh

# copy pod with a different image
kubectl debug <pod-name> -it --copy-to=debug-pod --set-image=<container-name>=ubuntu
```

### Node Debugging

Debug a node by creating a privileged pod with the node's filesystem mounted:

```sh
kubectl debug node/<node-name> -it --image=ubuntu
# node filesystem is at /host
```

## Debugging Services

Checklist for debugging service connectivity:

```sh
# 1. does the service exist?
kubectl get svc <service-name>

# 2. does the service have endpoints?
kubectl get endpoints <service-name>
kubectl get endpointslices -l kubernetes.io/service-name=<service-name>

# 3. do selectors match pod labels?
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels

# 4. is targetPort correct?
kubectl describe svc <service-name>

# 5. can you reach pod IPs directly?
kubectl exec <debug-pod> -- wget -qO- <pod-ip>:<port>

# 6. does DNS resolve?
kubectl exec <debug-pod> -- nslookup <service-name>
kubectl exec <debug-pod> -- nslookup <service-name>.<namespace>.svc.cluster.local

# 7. is kube-proxy running?
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

## Debugging Init Containers

Init containers run before app containers. If they fail, the pod stays in `Init:` state.

```sh
# check init container status — shows Init:0/1, Init:Error, etc.
kubectl get pod <pod-name>

# inspect init container details
kubectl describe pod <pod-name>

# check init container logs
kubectl logs <pod-name> -c <init-container-name>
```

## Useful Commands

```sh
# get all pods with their status
kubectl get pods -A -o wide

# get pod events sorted by timestamp
kubectl get events --sort-by='.lastTimestamp' --field-selector involvedObject.name=<pod-name>

# watch pod status changes
kubectl get pods -w

# get all pods in a non-running state
kubectl get pods -A --field-selector status.phase!=Running

# check resource usage
kubectl top pods
kubectl top pods --containers

# get pod YAML for inspection
kubectl get pod <pod-name> -o yaml

# delete and recreate a pod from YAML
kubectl replace --force -f <pod-definition>.yaml

# check all events in a namespace
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```
