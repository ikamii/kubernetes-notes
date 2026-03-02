# RBAC — Role-Based Access Control
Docs: [K8s Docs - RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

## Overview
- **Authentication** (authn) — who are you? (certificates, tokens, OIDC)
- **Authorization** (authz) — what can you do? (RBAC, ABAC, Webhook)
- RBAC is enabled by default: `--authorization-mode=RBAC` on the API server

### RBAC API Objects
| Object | Scope | Purpose |
|--------|-------|---------|
| Role | Namespace | Defines permissions within a namespace |
| ClusterRole | Cluster | Defines permissions cluster-wide or for non-namespaced resources |
| RoleBinding | Namespace | Grants a Role/ClusterRole to subjects in a namespace |
| ClusterRoleBinding | Cluster | Grants a ClusterRole to subjects cluster-wide |

## Roles & ClusterRoles

### Role (namespace-scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]          # "" = core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

### ClusterRole (cluster-scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "watch", "list"]
```

### Common Verbs
`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

### API Groups
- `""` — core group (pods, services, configmaps, secrets, nodes, namespaces, PVs, PVCs)
- `apps` — deployments, statefulsets, daemonsets, replicasets
- `batch` — jobs, cronjobs
- `networking.k8s.io` — networkpolicies, ingresses
- `rbac.authorization.k8s.io` — roles, rolebindings, clusterroles, clusterrolebindings
- `storage.k8s.io` — storageclasses

### Restrict to specific resource names
```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    resourceNames: ["my-pod"]   # only this specific pod
    verbs: ["get", "watch"]
```

## RoleBindings & ClusterRoleBindings

### RoleBinding — bind a Role to a user in a namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding — bind a ClusterRole to a ServiceAccount in a namespace
- A ClusterRole bound via a RoleBinding is scoped to the binding's namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: deploy-bot
    namespace: dev
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding — bind a ClusterRole cluster-wide
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## Checking Permissions

### kubectl auth can-i
```sh
# check if you can create pods
kubectl auth can-i create pods

# check if you can delete deployments in a specific namespace
kubectl auth can-i delete deployments -n production

# check permissions for another user (impersonation)
kubectl auth can-i create pods --as jane

# check permissions for a ServiceAccount
kubectl auth can-i list secrets --as system:serviceaccount:dev:deploy-bot

# list all permissions for current user
kubectl auth can-i --list

# list all permissions for a user in a namespace
kubectl auth can-i --list --as jane -n dev
```

## Useful Commands
```sh
# create a role imperatively
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev

# create a clusterrole imperatively
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes

# create a rolebinding imperatively
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n dev

# create a rolebinding for a ServiceAccount
kubectl create rolebinding deploy-binding --clusterrole=admin --serviceaccount=dev:deploy-bot -n dev

# create a clusterrolebinding imperatively
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --group=ops-team

# list roles and bindings
kubectl get roles,rolebindings -n dev
kubectl get clusterroles,clusterrolebindings

# describe a role to see its rules
kubectl describe role pod-reader -n dev
kubectl describe clusterrole cluster-admin
```

## Notes
> 1. You cannot edit the `roleRef` in a binding — delete and recreate it.
> 2. Built-in ClusterRoles: `cluster-admin`, `admin`, `edit`, `view` — use these as a starting point.
> 3. A RoleBinding can reference a ClusterRole but scopes it to the binding's namespace.
