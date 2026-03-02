# Helm & Kustomize
Docs: [Helm Docs](https://helm.sh/docs/) | [K8s Docs - Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)

---

## Helm
- Package manager for Kubernetes
- A **chart** is a collection of templates + default values that describe a set of K8s resources
- A **release** is an installed instance of a chart

### Repositories
```sh
# add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# update repository index
helm repo update

# list configured repositories
helm repo list

# search for charts in repos
helm search repo nginx

# search for charts on Artifact Hub
helm search hub wordpress
```

### Installing Charts
```sh
# install a chart with default values
helm install my-release bitnami/nginx

# install into a specific namespace (create if needed)
helm install my-release bitnami/nginx -n web --create-namespace

# install with custom values file
helm install my-release bitnami/nginx -f custom-values.yaml

# install with inline value overrides
helm install my-release bitnami/nginx --set replicaCount=3

# dry-run to see generated manifests without installing
helm install my-release bitnami/nginx --dry-run
```

### Upgrading & Rolling Back
```sh
# upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=5

# upgrade with a values file
helm upgrade my-release bitnami/nginx -f updated-values.yaml

# rollback to a previous revision
helm rollback my-release 1

# view release history
helm history my-release
```

### Uninstalling
```sh
# uninstall a release
helm uninstall my-release

# uninstall from a specific namespace
helm uninstall my-release -n web
```

### Inspecting Charts
```sh
# show default values for a chart
helm show values bitnami/nginx

# show chart metadata
helm show chart bitnami/nginx

# show the full chart info (README + values + chart.yaml)
helm show all bitnami/nginx

# list all releases
helm list

# list releases in all namespaces
helm list -A

# get details of a release
helm get values my-release
helm get manifest my-release
```

---

## Kustomize
- Built into `kubectl` — no extra installation needed
- Template-free customization of Kubernetes manifests
- Uses a `kustomization.yaml` file to define transformations

### Basic kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

# add a prefix to all resource names
namePrefix: staging-

# add labels to all resources
commonLabels:
  env: staging

# add annotations to all resources
commonAnnotations:
  managed-by: kustomize
```

### Applying Kustomize
```sh
# apply a kustomization directory
kubectl apply -k ./overlays/staging/

# preview the output without applying
kubectl kustomize ./overlays/staging/
```

### Image Overrides
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

images:
  - name: nginx
    newName: my-registry/nginx
    newTag: "1.25"
```

### ConfigMap & Secret Generators
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

configMapGenerator:
  - name: app-config
    literals:
      - DB_HOST=postgres
      - DB_PORT=5432

secretGenerator:
  - name: app-secret
    literals:
      - DB_PASSWORD=supersecret
```

### Patches
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

patches:
  - target:
      kind: Deployment
      name: nginx
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

### Bases & Overlays
- **Base**: shared resources (e.g., `base/deployment.yaml`, `base/service.yaml`)
- **Overlay**: environment-specific customizations that reference the base

```
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

Overlay `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

patches:
  - target:
      kind: Deployment
      name: nginx
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 10
```

## Useful Commands
```sh
# --- Helm ---
helm repo add <name> <url>
helm repo update
helm install <release> <chart> -n <namespace> --create-namespace
helm upgrade <release> <chart> --set key=value
helm rollback <release> <revision>
helm uninstall <release>
helm list -A
helm show values <chart>
helm history <release>

# --- Kustomize ---
kubectl apply -k <directory>
kubectl kustomize <directory>            # preview output
kubectl delete -k <directory>
```
