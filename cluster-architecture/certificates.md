# Certificates
Docs: [K8s Docs - PKI Certificates](https://kubernetes.io/docs/setup/best-practices/certificates/)

## Overview

- Kubernetes uses PKI (Public Key Infrastructure) for authentication between components
- All cluster certificates are stored in `/etc/kubernetes/pki/` on control plane nodes
- Key certificate pairs:

| Certificate | Purpose |
|-------------|---------|
| `ca.crt` / `ca.key` | Cluster CA — signs all other certificates |
| `apiserver.crt` | API server serving certificate |
| `apiserver-kubelet-client.crt` | API server → kubelet authentication |
| `etcd/ca.crt` | etcd CA (separate from cluster CA) |
| `front-proxy-ca.crt` | Front proxy CA for aggregation layer |

- kubeadm generates all certificates automatically during `kubeadm init`

---

## Certificate Signing Requests (CSR)
Docs: [K8s Docs - CSRs](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

### Create a certificate for a new user

**Step 1 — Generate a private key and CSR with openssl**
```sh
# generate a private key
openssl genrsa -out jane.key 2048

# create a certificate signing request
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr
```
- `CN` (Common Name) becomes the username
- `O` (Organization) becomes the group

**Step 2 — Create a CertificateSigningRequest object**
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: <base64-encoded-csr>        # cat jane.csr | base64 | tr -d '\n'
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
```
```sh
# encode the CSR and create the object
CSR=$(cat jane.csr | base64 | tr -d '\n')
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $CSR
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
EOF
```

**Step 3 — Approve (or deny) the CSR**
```sh
kubectl certificate approve jane
# or
kubectl certificate deny jane
```

**Step 4 — Extract the signed certificate**
```sh
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

**Step 5 — Use the certificate**
```sh
# set credentials in kubeconfig
kubectl config set-credentials jane --client-key=jane.key --client-certificate=jane.crt

# set context
kubectl config set-context jane-ctx --cluster=kubernetes --user=jane

# use the context
kubectl config use-context jane-ctx
```

---

## Checking Certificate Expiration

```sh
# check expiration of all kubeadm-managed certificates
kubeadm certs check-expiration

# inspect a specific certificate with openssl
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 Validity
```

- kubeadm certificates are valid for **1 year** by default
- The cluster CA is valid for **10 years**

---

## Renewing Certificates

```sh
# renew all kubeadm-managed certificates
kubeadm certs renew all

# renew a specific certificate
kubeadm certs renew apiserver

# restart control plane components after renewal
# (kubeadm static pods pick up new certs on restart)
crictl pods --name 'kube-apiserver-*' -q | xargs crictl rmp
```

- Running `kubeadm upgrade` automatically renews certificates
- Renew certificates **before** they expire to avoid cluster outage

---

## Useful Commands
```sh
# list all CSRs
kubectl get csr

# describe a CSR
kubectl describe csr <name>

# approve/deny a CSR
kubectl certificate approve <name>
kubectl certificate deny <name>

# delete a CSR
kubectl delete csr <name>

# check certificate expiration
kubeadm certs check-expiration

# renew all certificates
kubeadm certs renew all

# view certificate details
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```
