# CRDs & Operators
Docs: [K8s Docs - Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

## Custom Resource Definitions

### What is a CRD?
- Extends the Kubernetes API with new resource types
- Once a CRD is created, you can create/read/update/delete custom resources (CRs) using `kubectl`
- No need to modify or recompile the API server

### CRD Manifest
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.example.com    # must be <plural>.<group>
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                schedule:
                  type: string
                retentionDays:
                  type: integer
  scope: Namespaced                     # or Cluster
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
      - bk
```

### Creating a CRD
```sh
kubectl apply -f backup-crd.yaml

# verify the CRD is registered
kubectl get crd backups.stable.example.com
```

### Creating a Custom Resource
```yaml
apiVersion: stable.example.com/v1
kind: Backup
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  retentionDays: 30
```

```sh
kubectl apply -f daily-backup.yaml

# list custom resources
kubectl get backups
kubectl get bk          # using shortName

# describe a custom resource
kubectl describe backup daily-backup

# delete a custom resource
kubectl delete backup daily-backup
```

## Operators

### What is an Operator?
- A pattern that combines a CRD with a custom controller
- The controller watches for changes to custom resources and takes action (reconciliation loop)
- Encodes operational knowledge (install, upgrade, backup, scaling) into software

### How Operators Work
1. You define a CRD (e.g., `PostgresCluster`)
2. The operator's controller watches for `PostgresCluster` resources
3. When you create/update/delete a `PostgresCluster`, the controller reacts:
   - Creates Deployments, Services, ConfigMaps, PVCs as needed
   - Handles upgrades, failover, backups automatically

### Common Operators
- **cert-manager** — automates TLS certificate management
- **Prometheus Operator** — manages Prometheus monitoring stack
- **Strimzi** — manages Apache Kafka clusters

## Useful Commands
```sh
# list all CRDs in the cluster
kubectl get crd

# describe a CRD
kubectl describe crd <crd-name>

# list custom resources of a type
kubectl get <resource-name>

# delete a CRD (also deletes all its custom resources!)
kubectl delete crd <crd-name>

# check API resources (includes CRDs)
kubectl api-resources | grep <group>
```

## Notes
> 1. Deleting a CRD deletes all custom resources of that type — be careful.
> 2. CRDs alone don't do anything — you need a controller (operator) to act on the custom resources.
