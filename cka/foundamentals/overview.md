# CKA自学笔记6:Kubernetes集群概览

## 摘要

包含下面内容：

- [容器层](#container-layer)
- [Kubernetes层](#kubernetes-layer)

提示：

后续实验环境都是使用在阿里云部署的Ubuntu三节点集群，三个节点分别为 `cka001`、`cka002` 和 `cka003`。

## 容器层

场景：

使用Containerd服务，通过命令`nerdctl`来管理我们的镜像和容器，这与Docker的概念相同。

    * Get namespace.
    * Get containers.
    * Get images.
    * Get volumes.
    * Get overall status.
    * Get network status.

演示：

读取命名空间namespaces。

```bash
sudo nerdctl namespace ls
```

运行结果：

```console
NAME      CONTAINERS    IMAGES    VOLUMES    LABELS
k8s.io    21            30        0      
```

读取命名空间 `k8s.io`下的容器。

```bash
sudo nerdctl -n k8s.io ps
```

运行结果：

```console
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

读取命名空间 `k8s.io`下的镜像。

```bash
sudo nerdctl -n k8s.io image ls -a
```

读取命名空间 `k8s.io`下的卷Volume。初始化安装后，该命名空间下没有任何卷。

```bash
sudo nerdctl -n k8s.io volume ls
```

读取集群状态。

```bash
sudo nerdctl stats
```

读取网络状态。

```bash
sudo nerdctl network ls
sudo nerdctl network inspect bridge
sudo nerdctl network inspect k8s-pod-network
```

运行结果：

```console
NETWORK ID    NAME               FILE
              k8s-pod-network    /etc/cni/net.d/10-calico.conflist
0             bridge             /etc/cni/net.d/nerdctl-bridge.conflist
              host               
              none
```

Get network interface in host `cka001` with command `ip addr list`.

IP pool of `10.4.0.1/24` is `ipam` and defined in `/etc/cni/net.d/nerdctl-bridge.conflist`.

使用命令`ip addr list`获取主机`cka001`的网络接口。`10.4.0.1/24`的IP池是`ipam`，在`/etc/cni/net.d/nerdctl-bridge.conflist`中定义。

```console
lo                   : inet 127.0.0.1/8 qlen 1000
eth0                 : inet <cka001_ip>/24 brd xxx.xxx.xxx.255 scope global dynamic eth0
tunl0@NONE           : inet 10.244.228.192/32 scope global tunl0
cali96e32d88db2@if4  :
cali93613212490@if4  :
```

`nerdctl-bridge.conflist`文件的作用是：

- 定义了nerdctl使用的默认桥接CNI网络的配置，包括网络名称、子网、网关、IP分配策略等[1](https://github.com/containerd/nerdctl/blob/main/README.md) ，[2](https://github.com/containerd/nerdctl/blob/main/docs/cni.md)。
- 使得nerdctl可以使用docker run -it --rm alpine这样的命令来运行一个容器，并自动分配一个10.4.0.0/24网段的IP地址[1](https://github.com/containerd/nerdctl/blob/main/README.md)，[3](https://github.com/containerd/nerdctl)。
- 使得nerdctl可以支持一些基本的CNI插件，如bridge, portmap, firewall, tuning[1](https://github.com/containerd/nerdctl/blob/main/README.md)，[2](https://github.com/containerd/nerdctl/blob/main/docs/cni.md)。

## Kubernetes层

场景：
    * 节点Nodes
    * 命名空间Namespaces
    * 系统Pods

演示：

读取节点状态：

```bash
kubectl get node -o wide
```

在三个节点上有四个初始的命名空间。

```bash
kubectl get namespace -A
```

运行结果：

```bash
NAME              STATUS   AGE
default           Active   56m
kube-node-lease   Active   56m
kube-public       Active   56m
kube-system       Active   56m
```

在三个节点上的初始化Pod。

```bash
kubectl get pod -A -o wide
```

运行结果：

```console
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

总结： 
下面列出了初始集群中主节点和所有节点中所包含的容器和Pod的关系。 

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

参考：

- pause容器：[文章1](https://zhuanlan.zhihu.com/p/464712164) and [文章2](https://cloud.tencent.com/developer/article/1583919).

- [nerdctl](https://github.com/containerd/nerdctl)
