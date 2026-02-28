# CNI - Container Network Interface
Docs: [K8s Docs - Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

## Check CNI
```sh
cat /var/lib/kubelet/config.yaml | grep containerRuntimeEndpoint
```

## CNI supported plugins path
```sh
ls /opt/cni/bin
```

## Check configured CNI Plugin
```sh
ls /etc/cni/net.d/

# or check the pods

kubectl get pods -A
```

## Install CNI (ex. Calico)
Docs -> [Link](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

1. Install CRDs and Operators
    ```sh
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
    ```

2. Download Custom Resource manifest
    ```sh
    curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml -O
    ```

3. Edit `custom-resources.yaml` and update the cidr
    ```yaml
    apiVersion: operator.tigera.io/v1
    kind: Installation
    metadata:
    name: default
    spec:
    # Configures Calico networking.
    calicoNetwork:
        ipPools:
        - name: default-ipv4-ippool
            blockSize: 26
            cidr: x.x.x.x/16  # CHANGE ME
            encapsulation: VXLANCrossSubnet
            natOutgoing: Enabled
            nodeSelector: all()
    ```

4. Apply manifest
    ```sh
    kubectl apply -f custom-resources.yaml
    ```

## Delete CNI
After removing all k8s resources:

```sh
rm /etc/cni/net.d/<cni-conflist>
```

## Check Pod CIDR
```sh
kubectl cluster-info dump | grep -m 1 cluster-cidr

# or per-node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

### Notes
1. Not all CNI support `Network Policies` (Flannel does NOT, Calico and Weave do)

