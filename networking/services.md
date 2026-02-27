# Services

## Get IP Range for PODs
```sh
cat /etc/kubernetes/manifests/kube-controller-manager.yaml   | grep cluster-cidr
```

## Get IP Range for Services
```sh
cat /etc/kubernetes/manifests/kube-apiserver.yaml   | grep service-cluster-ip-range
```

