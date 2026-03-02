# API Deprecations
Docs: [K8s Docs - Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)

## Overview

- Kubernetes APIs follow a versioning scheme that indicates stability:

| Version | Stability | Example |
|---------|-----------|---------|
| `v1alpha1` | Alpha — disabled by default, may change or be removed | `admissionregistration.k8s.io/v1alpha1` |
| `v1beta1` | Beta — enabled by default, mostly stable | `flowcontrol.apiserver.k8s.io/v1beta1` |
| `v1` | Stable (GA) — production-ready, backward compatible | `apps/v1` |

- **Deprecation policy**: once a GA API version is released, the old beta version must be supported for at least **9 months or 3 releases** (whichever is longer)
- Deprecated APIs still function but print warnings; they are eventually **removed** in a future release
- Example: `extensions/v1beta1` Ingress was deprecated in v1.14, removed in v1.22 (replaced by `networking.k8s.io/v1`)

---

## Checking API Versions

```sh
# list all available API versions on the cluster
kubectl api-versions

# list all API resources with their group, version, and kind
kubectl api-resources

# list resources for a specific API group
kubectl api-resources --api-group=apps

# check the preferred version for a specific resource
kubectl explain ingress
# the output header shows the current GROUP and VERSION
```

---

## Identifying Deprecated APIs

```sh
# kubectl prints warnings when you use deprecated APIs
kubectl get ingress --api-version=extensions/v1beta1
# Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+

# check for deprecated API usage in existing manifests
kubectl get <resource> -o yaml | head -3   # check apiVersion field

# use --warnings-as-errors to fail on deprecated API usage
kubectl apply -f manifest.yaml --warnings-as-errors
```

- When upgrading clusters, audit all manifests and Helm charts for deprecated API versions **before** upgrading
- The `kubectl convert` plugin can help migrate manifests to newer API versions:
```sh
# install the convert plugin
kubectl krew install convert

# convert a manifest to a newer API version
kubectl convert -f old-ingress.yaml --output-version networking.k8s.io/v1
```

---

## Useful Commands
```sh
# list all API versions
kubectl api-versions

# list all API resources
kubectl api-resources

# list API resources with extra columns (shortnames, verbs)
kubectl api-resources -o wide

# check the API group/version for a resource
kubectl explain <resource>

# apply with warnings treated as errors
kubectl apply -f manifest.yaml --warnings-as-errors
```
