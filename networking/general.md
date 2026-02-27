# Explore

## Get nodes
```sh
kubectl get nodes
```

## Get Node IPs
```sh
kubectl get nodes -o wide
```

## Get network interface
```sh
ip a | grep <node-ip> -B 2
```

## Get default Gateway
```sh
ip route show default
```

## Get listening ports of control-plane components
```
netstat -nplt
```
