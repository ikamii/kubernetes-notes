# Useful kubectl Commands

## Aliases & Setup

```sh
# alias
alias k=kubectl

# autocomplete (bash)
source <(kubectl completion bash)
complete -F __start_kubectl k

# autocomplete (zsh)
source <(kubectl completion zsh)

# set default editor
export KUBE_EDITOR=vim,nano,micro
```

**Short resource names:**

| Short | Full Name               |
|-------|-------------------------|
| po    | pods                    |
| deploy| deployments             |
| svc   | services                |
| ns    | namespaces              |
| no    | nodes                   |
| rs    | replicasets             |
| ds    | daemonsets              |
| sts   | statefulsets            |
| cm    | configmaps              |
| sa    | serviceaccounts         |
| pv    | persistentvolumes       |
| pvc   | persistentvolumeclaims  |
| ing   | ingresses               |
| netpol| networkpolicies         |
| csr   | certificatesigningrequests |
| ep    | endpoints               |
| ev    | events                  |
| sc    | storageclasses          |
| pdb   | poddisruptionbudgets    |

## Dry Run & YAML Generation

Use `--dry-run=client -o yaml` to generate YAML without creating the resource.

```sh
# generate YAML and redirect to file
k run <pod-name> --image=<image> --dry-run=client -o yaml > pod.yaml

# generate deployment YAML
k create deployment <deployment-name> --image=<image> --dry-run=client -o yaml > deploy.yaml

# generate service YAML
k expose deployment <deployment-name> --port=<port> --dry-run=client -o yaml > svc.yaml
```

## Creating Resources (Imperative)

### kubectl run (Pods)

```sh
# basic pod
k run <pod-name> --image=<image>

# pod with labels
k run <pod-name> --image=<image> --labels="<key>=<value>,<key>=<value>"

# pod with port
k run <pod-name> --image=<image> --port=<port>

# pod with environment variables
k run <pod-name> --image=<image> --env="<key>=<value>" --env="<key>=<value>"

# pod with command and args
k run <pod-name> --image=<image> -- <command>

# pod with custom command
k run <pod-name> --image=<image> --command -- /bin/sh -c "<command>"

# pod with resource requests and limits
k run <pod-name> --image=<image> --requests="cpu=100m,memory=128Mi" --limits="cpu=200m,memory=256Mi"

# pod with service account
k run <pod-name> --image=<image> --serviceaccount=<service-account-name>

# temporary interactive pod (auto-removed on exit)
k run <pod-name> --image=<image> --rm -it -- /bin/sh

# generate YAML
k run <pod-name> --image=<image> --dry-run=client -o yaml > pod.yaml
```

### kubectl create deployment

```sh
# basic deployment
k create deployment <deployment-name> --image=<image>

# with replicas
k create deployment <deployment-name> --image=<image> --replicas=<count>

# with port
k create deployment <deployment-name> --image=<image> --port=<port>

# multiple containers
k create deployment <deployment-name> --image=<image-1> --image=<image-2>

# generate YAML
k create deployment <deployment-name> --image=<image> --dry-run=client -o yaml > deploy.yaml
```

### kubectl create service clusterip

```sh
# basic ClusterIP service
k create service clusterip <service-name> --tcp=<port>:<target-port>

# named port mapping
k create service clusterip <service-name> --tcp=<port>:<target-port>

# headless service (no cluster IP)
k create service clusterip <service-name> --tcp=<port>:<target-port> --clusterip="None"

# generate YAML
k create service clusterip <service-name> --tcp=<port>:<target-port> --dry-run=client -o yaml
```

### kubectl create service nodeport

```sh
# basic NodePort service
k create service nodeport <service-name> --tcp=<port>:<target-port>

# with specific node port
k create service nodeport <service-name> --tcp=<port>:<target-port> --node-port=<node-port>

# generate YAML
k create service nodeport <service-name> --tcp=<port>:<target-port> --dry-run=client -o yaml
```

### kubectl expose

```sh
# expose deployment as ClusterIP
k expose deployment <deployment-name> --port=<port> --target-port=<target-port> --type=ClusterIP

# expose deployment as NodePort
k expose deployment <deployment-name> --port=<port> --type=NodePort

# expose pod
k expose pod <pod-name> --port=<port> --target-port=<target-port> --name=<service-name>

# with custom service name
k expose deployment <deployment-name> --port=<port> --name=<service-name>

# generate YAML
k expose deployment <deployment-name> --port=<port> --dry-run=client -o yaml > svc.yaml
```

### kubectl create configmap

```sh
# from literal values
k create configmap <configmap-name> --from-literal=<key>=<value> --from-literal=<key>=<value>

# from file
k create configmap <configmap-name> --from-file=<file>

# from file with custom key
k create configmap <configmap-name> --from-file=<key>=<file>

# from env file
k create configmap <configmap-name> --from-env-file=<file>

# from directory
k create configmap <configmap-name> --from-file=<directory>/

# generate YAML
k create configmap <configmap-name> --from-literal=<key>=<value> --dry-run=client -o yaml
```

### kubectl create secret

```sh
# generic from literals
k create secret generic <secret-name> --from-literal=<key>=<value> --from-literal=<key>=<value>

# generic from file
k create secret generic <secret-name> --from-file=<key>=<file>

# TLS secret
k create secret tls <secret-name> --cert=<cert-file> --key=<key-file>

# docker registry secret
k create secret docker-registry <secret-name> \
  --docker-server=<server> \
  --docker-username=<username> \
  --docker-password=<password>

# generate YAML
k create secret generic <secret-name> --from-literal=<key>=<value> --dry-run=client -o yaml
```

### kubectl create namespace

```sh
# create namespace
k create namespace <namespace>
```

### kubectl create serviceaccount

```sh
# create service account
k create serviceaccount <service-account-name>

# in specific namespace
k create serviceaccount <service-account-name> -n <namespace>
```

### kubectl create job

```sh
# basic job
k create job <job-name> --image=<image> -- <command>

# create job from existing cronjob
k create job <job-name> --from=cronjob/<cronjob-name>
```

### kubectl create cronjob

```sh
# basic cronjob
k create cronjob <cronjob-name> --image=<image> --schedule="<cron-expression>" -- <command>
```

### kubectl create ingress

```sh
# single rule
k create ingress <ingress-name> --rule="<host>/<path>=<service-name>:<port>"

# with TLS
k create ingress <ingress-name> --rule="<host>/<path>=<service-name>:<port>,tls=<tls-secret-name>"

# multiple rules
k create ingress <ingress-name> --rule="<host-1>/=<service-name-1>:<port>" --rule="<host-2>/=<service-name-2>:<port>"

# with default backend
k create ingress <ingress-name> --default-backend=<service-name>:<port>

# with annotation
k create ingress <ingress-name> --rule="<host>/=<service-name>:<port>" \
  --annotation=<key>=<value>
```

### kubectl create role

```sh
# basic role
k create role <role-name> --verb=get,list,watch --resource=pods

# with specific resource names
k create role <role-name> --verb=get --resource=pods --resource-name=<resource-name>

# multiple resources
k create role <role-name> --verb=get,list --resource=pods,services

# with API group
k create role <role-name> --verb=get --resource=deployments.apps
```

### kubectl create rolebinding

```sh
# bind to user
k create rolebinding <rolebinding-name> --role=<role-name> --user=<username>

# bind to group
k create rolebinding <rolebinding-name> --role=<role-name> --group=<group-name>

# bind to service account
k create rolebinding <rolebinding-name> --role=<role-name> --serviceaccount=<namespace>:<service-account-name>
```

### kubectl create clusterrole

```sh
# basic cluster role
k create clusterrole <clusterrole-name> --verb=get,list,watch --resource=pods

# non-resource URLs
k create clusterrole <clusterrole-name> --verb=get --non-resource-url=<url>
```

### kubectl create clusterrolebinding

```sh
# bind to user
k create clusterrolebinding <clusterrolebinding-name> --clusterrole=<clusterrole-name> --user=<username>

# bind to service account
k create clusterrolebinding <clusterrolebinding-name> --clusterrole=<clusterrole-name> --serviceaccount=<namespace>:<service-account-name>
```

### kubectl create quota

```sh
# basic quota
k create quota <quota-name> --hard=pods=10,requests.cpu=4,requests.memory=8Gi

# with limits
k create quota <quota-name> --hard=pods=10,limits.cpu=8,limits.memory=16Gi
```

### kubectl create pdb

```sh
# with min-available
k create pdb <pdb-name> --selector=<key>=<value> --min-available=<count>

# with max-unavailable
k create pdb <pdb-name> --selector=<key>=<value> --max-unavailable=<count>
```

### kubectl create token

```sh
# create token for service account
k create token <service-account-name>

# with duration
k create token <service-account-name> --duration=<duration>
```

## Updating Resources

### kubectl set

```sh
# update container image
k set image deployment/<deployment-name> <container-name>=<image>:<tag>

# set environment variable
k set env deployment/<deployment-name> <key>=<value>

# set env from configmap
k set env deployment/<deployment-name> --from=configmap/<configmap-name>

# set env from secret
k set env deployment/<deployment-name> --from=secret/<secret-name>

# set resource requests and limits
k set resources deployment/<deployment-name> --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi

# set service account
k set serviceaccount deployment/<deployment-name> <service-account-name>
```

### kubectl scale

```sh
# scale deployment
k scale deployment <deployment-name> --replicas=<count>

# scale replicaset
k scale rs <replicaset-name> --replicas=<count>

# scale statefulset
k scale statefulset <statefulset-name> --replicas=<count>

# conditional scale (only if current replicas match)
k scale deployment <deployment-name> --replicas=<count> --current-replicas=<current-count>
```

### kubectl rollout

```sh
# check rollout status
k rollout status deployment/<deployment-name>

# view rollout history
k rollout history deployment/<deployment-name>

# view specific revision
k rollout history deployment/<deployment-name> --revision=<revision>

# undo last rollout
k rollout undo deployment/<deployment-name>

# undo to specific revision
k rollout undo deployment/<deployment-name> --to-revision=<revision>

# restart rollout (triggers rolling update)
k rollout restart deployment/<deployment-name>

# pause rollout
k rollout pause deployment/<deployment-name>

# resume rollout
k rollout resume deployment/<deployment-name>
```

### kubectl autoscale

```sh
# autoscale deployment
k autoscale deployment <deployment-name> --min=<min> --max=<max> --cpu-percent=<percent>
```

### kubectl label

```sh
# add label
k label pod <pod-name> <key>=<value>

# overwrite existing label
k label pod <pod-name> <key>=<value> --overwrite

# remove label
k label pod <pod-name> <key>-

# label a node
k label node <node-name> <key>=<value>

# label all pods
k label pods --all <key>=<value>
```

### kubectl annotate

```sh
# add annotation
k annotate pod <pod-name> <key>="<value>"

# remove annotation
k annotate pod <pod-name> <key>-
```

### kubectl taint

```sh
# add taint with NoSchedule effect
k taint node <node-name> <key>=<value>:NoSchedule

# add taint with NoExecute effect
k taint node <node-name> <key>=<value>:NoExecute

# add taint with PreferNoSchedule effect
k taint node <node-name> <key>=<value>:PreferNoSchedule

# remove taint
k taint node <node-name> <key>=<value>:NoSchedule-

# remove all taints with a key
k taint node <node-name> <key>-
```

### kubectl patch

```sh
# JSON strategic merge patch
k patch deployment <deployment-name> -p '{"spec":{"replicas":<count>}}'

# merge type patch
k patch deployment <deployment-name> --type=merge -p '{"spec":{"replicas":<count>}}'

# JSON patch type (add, remove, replace operations)
k patch deployment <deployment-name> --type=json -p '[{"op":"replace","path":"/spec/replicas","value":<count>}]'

# patch a node (e.g., remove a taint)
k patch node <node-name> --type=json -p '[{"op":"remove","path":"/spec/taints/0"}]'
```

## Apply / Edit / Delete / Replace

```sh
# apply from file
k apply -f <file>

# apply from directory
k apply -f <directory>/

# apply from URL
k apply -f <url>

# edit resource in-place
k edit deployment <deployment-name>

# edit in specific namespace
k edit deployment <deployment-name> -n <namespace>

# delete resource
k delete pod <pod-name>

# force delete (immediate)
k delete pod <pod-name> --force --grace-period 0

# delete by file
k delete -f <file>

# delete all pods in namespace
k delete pods --all -n <namespace>

# delete by label
k delete pods -l <key>=<value>

# replace resource (delete + recreate)
k replace -f <file>

# force replace (useful for immutable fields)
k replace --force -f <file>
```

## Viewing & Inspecting Resources

### kubectl get

```sh
# wide output
k get pods -o wide

# YAML output
k get pod <pod-name> -o yaml

# JSON output
k get pod <pod-name> -o json

# all namespaces
k get pods -A
k get pods --all-namespaces

# show labels
k get pods --show-labels

# label selector (equality-based)
k get pods -l <key>=<value>
k get pods -l <key>=<value>,<key>=<value>
k get pods -l <key>!=<value>

# label selector (set-based)
k get pods -l '<key> in (<value-1>,<value-2>)'
k get pods -l '<key> notin (<value>)'
k get pods -l '!<key>'

# field selector
k get pods --field-selector=status.phase=Running
k get pods --field-selector=metadata.name=<pod-name>

# sort by
k get pods --sort-by=.metadata.creationTimestamp
k get pods --sort-by=.status.containerStatuses[0].restartCount

# get multiple resources
k get pods,svc,deploy

# watch for changes
k get pods -w
```

### kubectl describe

```sh
# describe pod
k describe pod <pod-name>

# describe node
k describe node <node-name>

# describe service
k describe svc <service-name>

# describe all pods
k describe pods
```

### kubectl logs

```sh
# view logs
k logs <pod-name>

# follow logs (stream)
k logs -f <pod-name>

# previous container logs (after restart)
k logs <pod-name> --previous

# specific container in multi-container pod
k logs <pod-name> -c <container-name>

# tail last N lines
k logs <pod-name> --tail=<count>

# logs since time duration
k logs <pod-name> --since=<duration>

# logs by label selector
k logs -l <key>=<value>
```

### kubectl exec

```sh
# run a command
k exec <pod-name> -- <command>

# interactive shell
k exec -it <pod-name> -- /bin/sh

# specific container in multi-container pod
k exec -it <pod-name> -c <container-name> -- /bin/sh
```

### kubectl port-forward

```sh
# forward pod port
k port-forward pod/<pod-name> <local-port>:<pod-port>

# forward service port
k port-forward svc/<service-name> <local-port>:<service-port>
```

### kubectl get events

```sh
# get events sorted by time
k get events --sort-by=.metadata.creationTimestamp

# events for a specific resource
k get events --field-selector=involvedObject.name=<resource-name>
```

### kubectl explain

```sh
# explain a resource
k explain <resource>

# explain a specific field
k explain <resource>.<field>

# recursive (show all fields)
k explain <resource>.<field> --recursive

# specific API version
k explain <resource> --api-version=<api-version>
```

## Cluster & Node Operations

```sh
# cluster info
k cluster-info

# list API resources (with short names)
k api-resources

# namespaced resources only
k api-resources --namespaced=true

# cluster-scoped resources only
k api-resources --namespaced=false

# list API versions
k api-versions

# top nodes (resource usage)
k top nodes

# top pods
k top pods

# top pods sorted by cpu
k top pods --sort-by=cpu

# top pods sorted by memory
k top pods --sort-by=memory

# top pods with containers
k top pods --containers

# mark node as unschedulable
k cordon <node-name>

# mark node as schedulable
k uncordon <node-name>

# drain node (evict pods for maintenance)
k drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

## Config & Context

```sh
# show current context
k config current-context

# list all contexts
k config get-contexts

# switch context
k config use-context <context-name>

# set default namespace for current context
k config set-context --current --namespace=<namespace>

# view full config
k config view

# view config for current context only
k config view --minify
```

## RBAC

```sh
# check if you can perform an action
k auth can-i <verb> <resource>

# check as a specific user
k auth can-i <verb> <resource> --as=<username>

# check as a service account
k auth can-i <verb> <resource> --as=system:serviceaccount:<namespace>:<service-account-name>

# check all permissions
k auth can-i --list

# check in specific namespace
k auth can-i <verb> <resource> -n <namespace>

# show current user info
k auth whoami
```

## Certificates

```sh
# list certificate signing requests
k get csr

# approve a CSR
k certificate approve <csr-name>

# deny a CSR
k certificate deny <csr-name>

# delete a CSR
k delete csr <csr-name>
```

## JSONPath & Custom Columns

### JSONPath

```sh
# basic field
k get pods -o jsonpath='{.items[*].metadata.name}'

# multiple fields with range
k get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# specific index
k get pods -o jsonpath='{.items[0].metadata.name}'

# filter with conditional
k get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# node internal IPs
k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# all container images
k get pods -o jsonpath='{.items[*].spec.containers[*].image}'
```

### Custom Columns

```sh
# inline custom columns
k get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# from file
k get pods -o custom-columns-file=<file>

# nodes with taints
k get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```
