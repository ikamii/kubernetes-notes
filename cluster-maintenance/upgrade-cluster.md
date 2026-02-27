# Upgrade Cluster

## Get k8s version

```shell
kubectl get nodes
```

## Check the taints

```shell
kubectl describe nodes | grep -E "Name:|Taints:"
```

## Get the latest version available to upgrade for this major version

```shell
kubeadm upgrade plan

# For example for v1.33.0 -> v1.33.8
```

## Upgrading

### Control plane

### 1 . Drain the node
```shell
kubectl drain <control-plane-name> --ignore-daemonsets
```

### 2. Upgrade major version

1. Update keyring [kubernetes-docs](https://v1-34.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used):
   ```shell
   nano /etc/apt/sources.list.d/kubernetes.list
   
   # and change the version like below
   
   deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
   ```

2. Upgrade kubeadm on control plane
   ```shell
   sudo apt update
   sudo apt-cache madison kubeadm  # Check the available versions

   sudo apt-get install kubeadm=1.34.0-1.1

   # Verify version
   kubeadm version
   ```

3. Plan & Upgrade
   ```shell
   kubeadm upgrade plan v1.34.0

   kubeadm upgrade apply v1.34.0
   ```

4. Upgrade kubelet
   ```shell
   sudo apt-get install kubelet=1.34.0-1.1
   ```

5. Restart daemon and kubelet
   ```shell
      sudo systemctl daemon-reload
      
      sudo systemctl restart kubelet
   ```

6. Mark controlplane -> Schedulable
   ```shell
   kubectl uncordon <control-plane-name> 
   ``` 
  
#### Worker Node
>From now on, run commands on worker-node (ssh connection)
1. Drain the node
   ```shell
   kubectl drain <worker-node-name> --ignore-dameonsets
   ```

2. Update keyring [kubernetes-docs](https://v1-34.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used):
   ```shell
   nano /etc/apt/sources.list.d/kubernetes.list
   
   # and change the version like below
   
   deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
   ```

 3. Upgrade kubeadm on control plane
      ```shell
      sudo apt update
      sudo apt-cache madison kubeadm  # Check the available versions

      sudo apt-get install kubeadm=1.34.0-1.1

      # Verify version
      kubeadm version
      ```

4. Upgrade node
   ```shell
   kubeadm upgrade node
   ```

5. Upgrade kubelet
   ```shell 
   sudo apt-get install kubelet=1.34.0-1.1
   ```

6. Restart daemon and kubelet
   ```shell
   sudo systemctl daemon-reload
   
   sudo systemctl restart kubelet
   ```

7. Mark woker node -> Schedulable
   ```shell
   # On control-plane node

   kubectl uncordon <worker-node-name> 
   ``` 

  

## Notes

> 1. Upgrade the Nodes one by one.

> 2. If a pod does not belong to any replica set, it can't be evicted while draning the node (Draining fails). --force flag can be used but the pod is deleted forever.
> 
>       Mark the node unschedulable to avoid this use -> `kubectl cordon <node-name>`

