# Configuration
Docs: [K8s Docs - Configuration](https://kubernetes.io/docs/concepts/configuration/)

## ConfigMaps
Docs: [K8s Docs - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

- Store non-confidential configuration data as key-value pairs
- Can be consumed as environment variables or mounted as files

### Create a ConfigMap

**Imperative:**
```sh
# from literal values
kubectl create configmap app-config --from-literal=DB_HOST=postgres --from-literal=DB_PORT=5432

# from a file
kubectl create configmap app-config --from-file=config.properties

# from an env file
kubectl create configmap app-config --from-env-file=app.env
```

**Declarative:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: postgres
  DB_PORT: "5432"
  app.properties: |
    server.port=8080
    log.level=info
```

### Use as environment variables
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      # inject all keys as env vars
      envFrom:
        - configMapRef:
            name: app-config
      # or inject specific keys
      env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
```

### Use as volume mount
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```
- Each key in the ConfigMap becomes a file in `/etc/config/`
- Volume-mounted ConfigMaps are updated automatically (eventually)

---

## Secrets
Docs: [K8s Docs - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

- Store sensitive data (passwords, tokens, keys)
- Values are base64-encoded (not encrypted by default)

### Secret Types
| Type | Usage |
|------|-------|
| `Opaque` (default) | Arbitrary key-value data |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |

### Create a Secret

**Imperative:**
```sh
# from literal values (Opaque type)
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=s3cret

# TLS secret
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key

# Docker registry secret
kubectl create secret docker-registry my-reg \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

**Declarative:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=          # echo -n "admin" | base64
  password: czNjcmV0          # echo -n "s3cret" | base64
```

### Base64 encode/decode
```sh
# encode
echo -n "admin" | base64

# decode
echo "YWRtaW4=" | base64 --decode
```

### Use as environment variables
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - secretRef:
            name: db-secret
      # or specific keys
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

### Use as volume mount
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-secret
```

---

## SecurityContext
Docs: [K8s Docs - Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

- Defines privilege and access control settings for pods and containers

### Pod-level SecurityContext
- Applies to all containers in the pod
```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
```

### Container-level SecurityContext
- Overrides pod-level settings for a specific container
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        runAsUser: 1000
        runAsNonRoot: true
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          add: ["NET_BIND_SERVICE"]
          drop: ["ALL"]
```

### Common fields
| Field | Level | Description |
|-------|-------|-------------|
| `runAsUser` | Pod/Container | UID to run the process as |
| `runAsGroup` | Pod/Container | GID to run the process as |
| `runAsNonRoot` | Pod/Container | Fail if container runs as root |
| `fsGroup` | Pod | GID applied to all mounted volumes |
| `readOnlyRootFilesystem` | Container | Mount root filesystem as read-only |
| `allowPrivilegeEscalation` | Container | Prevent child process from gaining more privileges |
| `capabilities` | Container | Add or drop Linux capabilities |

---

## ServiceAccounts
Docs: [K8s Docs - ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)

- Provide an identity for processes running in pods
- Each namespace has a `default` ServiceAccount
- Pods use the `default` SA unless a different one is specified

### Create a ServiceAccount
```sh
kubectl create serviceaccount deploy-bot -n dev
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-bot
  namespace: dev
```

### Assign to a pod
```yaml
spec:
  serviceAccountName: deploy-bot
  automountServiceAccountToken: true    # default is true; set false to disable token mount
  containers:
    - name: app
      image: myapp:1.0
```

### Token projection
- Kubernetes automatically mounts a projected token at `/var/run/secrets/kubernetes.io/serviceaccount/`
- The token is audience-bound and time-limited (auto-rotated)
- Files mounted: `token`, `ca.crt`, `namespace`

---

## Useful Commands
```sh
# --- ConfigMaps ---
kubectl create configmap <name> --from-literal=key=value
kubectl get configmaps
kubectl describe configmap <name>

# --- Secrets ---
kubectl create secret generic <name> --from-literal=key=value
kubectl get secrets
kubectl describe secret <name>
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 --decode

# --- SecurityContext ---
# check which user a container is running as
kubectl exec <pod-name> -- id
kubectl exec <pod-name> -- whoami

# --- ServiceAccounts ---
kubectl create serviceaccount <name>
kubectl get serviceaccounts
kubectl describe serviceaccount <name>
```
