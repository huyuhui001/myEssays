# Cluster Overview

## Contents

- [Container Layer](#container-layer)
- [Kubernetes Layer](#kubernetes-layer)

!!! Information
    For environment setup, refer to [Installation on Aliyun Ubuntu](../installation/aliyun-ubuntu.md)

## Container Layer

!!! Scenario
    Use Containerd service to manage our images and containers via command `nerdctl`, which is same concept with Docker.

    * Get namespace.
    * Get containers.
    * Get images.
    * Get volumes.
    * Get overall status.
    * Get network status.

Demo: 

Get namespaces.
```console
sudo nerdctl namespace ls
```
Result
```
NAME      CONTAINERS    IMAGES    VOLUMES    LABELS
k8s.io    21            30        0      
```

Get containers under specific namespace `k8s.io`.
```console
sudo nerdctl -n k8s.io ps
```
Result
```
CONTAINER ID    IMAGE                                                                      COMMAND                   CREATED         STATUS    PORTS    NAMES
0a3625f22f65    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/coredns-74586cf9b6-4jwmk
121af2ecd1a1    registry.aliyuncs.com/google_containers/coredns:v1.8.6                     "/coredns -conf /etc…"    16 hours ago    Up                 k8s://kube-system/coredns-74586cf9b6-c5mll/coredns
49f6c7e3efe5    registry.aliyuncs.com/google_containers/kube-proxy:v1.24.0                 "/usr/local/bin/kube…"    16 hours ago    Up                 k8s://kube-system/kube-proxy-dmj2t/kube-proxy
4bba5fbd701d    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/kube-scheduler-cka001
57d47b57eb12    docker.io/calico/node:v3.23.3                                              "start_runit"             16 hours ago    Up                 k8s://kube-system/calico-node-w8nvl/calico-node
5ce4c351a886    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/calico-node-w8nvl
6456eef784bf    registry.aliyuncs.com/google_containers/kube-scheduler:v1.24.0             "kube-scheduler --au…"    16 hours ago    Up                 k8s://kube-system/kube-scheduler-cka001/kube-scheduler
6a687305871c    registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0             "kube-apiserver --ad…"    16 hours ago    Up                 k8s://kube-system/kube-apiserver-cka001/kube-apiserver
7dcb24568574    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/coredns-74586cf9b6-c5mll
a06b101118b8    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/kube-controller-manager-cka001
a07ef8c3fc3a    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/etcd-cka001
b8566d3e4174    registry.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0    "kube-controller-man…"    16 hours ago    Up                 k8s://kube-system/kube-controller-manager-cka001/kube-controller-manager
ca6ac26314ff    registry.aliyuncs.com/google_containers/coredns:v1.8.6                     "/coredns -conf /etc…"    16 hours ago    Up                 k8s://kube-system/coredns-74586cf9b6-4jwmk/coredns
cdc041b4791e    registry.aliyuncs.com/google_containers/etcd:3.5.3-0                       "etcd --advertise-cl…"    16 hours ago    Up                 k8s://kube-system/etcd-cka001/etcd
e0c59abadf2e    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/kube-proxy-dmj2t
e0d2e5f6ccff    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  16 hours ago    Up                 k8s://kube-system/kube-apiserver-cka001
```


Get images.
```console
sudo nerdctl -n k8s.io image ls -a
```

Get volumes. After inintial installation, no volume within namespaces.
```console
sudo nerdctl -n k8s.io volume ls
```

Get overall status
```console
sudo nerdctl stats
```

Get network status.
```console
sudo nerdctl network ls
sudo nerdctl network inspect bridge
sudo nerdctl network inspect k8s-pod-network
```
Result
```
NETWORK ID    NAME               FILE
              k8s-pod-network    /etc/cni/net.d/10-calico.conflist
0             bridge             /etc/cni/net.d/nerdctl-bridge.conflist
              host               
              none
```

Get network interface in host `cka001` with command `ip addr list`.

IP pool of `10.4.0.1/24` is `ipam` and defined in `/etc/cni/net.d/nerdctl-bridge.conflist`.
```
lo                   : inet 127.0.0.1/8 qlen 1000
eth0                 : inet <cka001_ip>/24 brd xxx.xxx.xxx.255 scope global dynamic eth0
tunl0@NONE           : inet 10.244.228.192/32 scope global tunl0
cali96e32d88db2@if4  :
cali93613212490@if4  :
```





## Kubernetes Layer

!!! Scenario
    * Nodes
    * Namespaces
    * System Pods

Demo:

!!! information
    In the demo, there are three nodes, `cka001`, `cka002`, and `cka003`.

Get nodes status.
```console
kubectl get node -o wide
```

There are four initial namespaces across three nodes.
```console
kubectl get namespace -A
```
Result
```
NAME              STATUS   AGE
default           Active   56m
kube-node-lease   Active   56m
kube-public       Active   56m
kube-system       Active   56m
```

There are some initial pods. 
```console
kubectl get pod -A -o wide
```
Result
```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE   NODE     NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-555bc4b957-l8bn2   1/1     Running   0          15h   cka003   <none>           <none>
kube-system   calico-node-255pc                          1/1     Running   0          15h   cka003   <none>           <none>
kube-system   calico-node-7tmnb                          1/1     Running   0          15h   cka002   <none>           <none>
kube-system   calico-node-w8nvl                          1/1     Running   0          15h   cka001   <none>           <none>
kube-system   coredns-74586cf9b6-4jwmk                   1/1     Running   0          15h   cka001   <none>           <none>
kube-system   coredns-74586cf9b6-c5mll                   1/1     Running   0          15h   cka001   <none>           <none>
kube-system   etcd-cka001                                1/1     Running   0          15h   cka001   <none>           <none>
kube-system   kube-apiserver-cka001                      1/1     Running   0          15h   cka001   <none>           <none>
kube-system   kube-controller-manager-cka001             1/1     Running   0          15h   cka001   <none>           <none>
kube-system   kube-proxy-dmj2t                           1/1     Running   0          15h   cka001   <none>           <none>
kube-system   kube-proxy-n77zw                           1/1     Running   0          15h   cka002   <none>           <none>
kube-system   kube-proxy-qs6rf                           1/1     Running   0          15h   cka003   <none>           <none>
kube-system   kube-scheduler-cka001                      1/1     Running   0          15h   cka001   <none>           <none>
```

!!! Summary 
    Below shows the relationship between containers and pods. 
    
    * Master node:
        * CoreDNS: 2 Pod
        * etcd: 1 Pod
        * apiserver: 1 Pod
        * controller-manager: 1 Pod
        * scheduler: 1 Pod
        * Calico Controller: 1 Pod
    * All nodes:
        * Calico Node: 1 Pod each
        * Proxy: 1 Pod each



!!! Reference
    - Container pause: [article](https://zhuanlan.zhihu.com/p/464712164) and [artical](https://cloud.tencent.com/developer/article/1583919).
    - [nerdctl](https://github.com/containerd/nerdctl)








