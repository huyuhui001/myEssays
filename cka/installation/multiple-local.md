# # CKA自学笔记2:多节点虚拟机安装Kubernetes

## 摘要

在本地Windows环境中，通过VMWare安装三台Ubuntu虚拟机。在Ubuntu虚拟机中安装基于Containerd的Kubernetes系统，并分别配置一个主节点Master和两个工作节点Worker。

## 本地虚拟机设置

VMWare 设置

* VMnet1: host-only模式, 网络subnet: 192.168.150.0/24
* VMnet8: NAT模式, 网络subnet: 11.0.1.0/24

通过VMWare创建客户虚拟机。

* 内存：4 GB
* CPU：1 CPUs with 2 Cores
* 操作系统：Ubuntu Server 22.04
* 网络：NAT

提示：

当前练习中，Kubernetes是基于Containerd，不是Docker。

## Ubuntu预配置

注意：下面的任务，需要在每台虚拟机中执行一次。以下简称虚拟机为节点。

在所有节点中创建用户`vagrant`。

```bash
sudo adduser vagrant
sudo usermod -aG adm,sudo,syslog,cdrom,dip,plugdev,lxd,root vagrant
sudo passwd vagrant
```

在所有节点中设置用户`root`的密码。

```bash
sudo passwd root
```

修改ssh服务的配置文件。开放`root`用户通过ssh登录（默认是禁用的）。

```bash
sudo vi /etc/ssh/sshd_config
```

把参数 `PermitRootLogin`的值从`prohibit-password` 改为`yes`。

```
PermitRootLogin yes
#PermitRootLogin prohibit-password
```

重新启动sshd服务。

```bash
sudo systemctl restart sshd
```

更改主机名，这里是`ubu1`.

```bash
sudo hostnamectl set-hostname ubu1
sudo hostnamectl set-hostname ubu1 --pretty
```

验证主机名是否被正确修改了，比如改为`ubu1`。

```bash
cat /etc/machine-info
```

验证主机名是否被正确修改了，比如改为`ubu1`。

```bash
cat /etc/hostname
```

验证主机IP地址`127.0.1.1` 已经配置给当前节点，比如`ubu1`。同时，在所有节点的 `/etc/hosts`文件中添加其他节点的IP和主机对应信息。

```bash
sudo vi /etc/hosts
```

以当前练习为例，修改后的`/etc/hosts`文件类似如下内容。

```
127.0.1.1 ubu1
11.0.1.129 ubu1
11.0.1.130 ubu2
11.0.1.131 ubu3
11.0.1.132 ubu4
```

创建文件`/etc/netplan/00-installer-config.yaml`。

```bash
sudo vi /etc/netplan/00-installer-config.yaml
```

更新此文件，设定当前节点使用固定IP地址，比如，`11.0.1.129`。

```yaml
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses:
      - 11.0.1.129/24
      nameservers:
        addresses:
        - 11.0.1.2
      routes:
      - to: default
        via: 11.0.1.2
  version: 2
```

执行下面命令时，使上述改动生效。注意，当前ssh连接会因此而断开。

```bash
sudo netplan apply
```

在所有节点禁用交换分区swap和防火墙firewall。

```bash
sudo swapoff -a
sudo ufw disable
sudo ufw status verbose
```

在所有节点的文件 `/etc/fstab`中注释掉涉及swap的那一行，修改后需要重启当前节点。

```bash
sudo vi /etc/fstab
```

修改后的结果类似如下。

```
/dev/disk/by-uuid/df370d2a-83e5-4895-8c7f-633f2545e3fe / ext4 defaults 0 1
# /swap.img     none    swap    sw      0       0
```

在所有节点设置统一的时区。

```bash
sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

sudo cp /etc/profile /etc/profile.bak
echo 'LANG="en_US.UTF-8"' | sudo tee -a /etc/profile

source /etc/profile
```

执行命令 `ll /etc/localtime`来验证时区是否修改正确。

```
lrwxrwxrwx 1 root root 33 Jul 15 22:00 /etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai
```

在所有节点修改内核设置。

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

手动将需要的这2个模块载入内核。

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

在所有节点修改网络设置。

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

生效上述修改。

```bash
sudo sysctl --system
```

注意：

重启当前节点，节点重启后，用账号`vagrant` 做如下验证，确保上述修改已生效。

* IP地址。
  
  ```bash
    ip addr list
  ```

* 主机名。
  
  ```bash
  cat /etc/machine-info
  cat /etc/hostname
  hostname
  ```

* 防火墙。
  
  ```bash
  sudo ufw status verbose
  ```

* 内核。
  
  ```bash
  lsmod | grep -i overlay
  lsmod | grep -i br_netfilter
  ```

* 网络。
  
  ```bash
  sudo sysctl -a | grep -i net.bridge.bridge-nf-call-ip*
  sudo sysctl -a | grep -i net.ipv4.ip_forward
  ```

## 安装Containerd

在所有节点上安装Containerd服务。

备份Ubuntu安装源的原文件。

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

安装Containered。

```bash
sudo apt-get update && sudo apt-get install -y containerd
```

修改文件`/etc/containerd/config.toml`来配置Contanerd服务，如果没有，就创建这个文件。

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

更新`sandbox_image`的值为`"registry.aliyuncs.com/google_containers/pause:3.6"`，以使用国内阿里云的源。
更新`SystemdCgroup ` 的值为 `true`，以使用Cgroup。

```
[plugins]
  [plugins."io.containerd.gc.v1.scheduler"]

  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"

    [plugins."io.containerd.grpc.v1.cri".cni]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```

重启Containerd 服务。

```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

## 安装nerdctl

在所有节点上安装nerdctl服务。

[`nerdctl`](https://github.com/containerd/nerdctl) 服务支持Contanerd所提供的容器化特性，特别是一些Docker不具备的新特性。

二进制安装包可以通过这个链接取得: https://github.com/containerd/nerdctl/releases 。

```bash
wget https://github.com/containerd/nerdctl/releases/download/v0.22.2/nerdctl-0.22.2-linux-amd64.tar.gz
tar -zxvf nerdctl-0.22.2-linux-amd64.tar.gz
sudo cp nerdctl /usr/bin/
```

验证`nerdctl`服务。

```bash
sudo nerdctl --help
```

列出初始安装Kubernetes时的容器container列表。

```bash
sudo nerdctl -n k8s.io ps
```

## 安装Kubernetes

在所有节点上安装Kubeadm，kubectl，kubelet。

安装和升级Ubuntu系统依赖包。

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt-get update
sudo apt-get install ebtables
sudo apt-get install libxtables12
sudo apt-get upgrade iptables
```

安装gpg证书。

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
```

添加Kubernetes安装源。 

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

检查当前`kubeadm`的版本。

```bash
apt policy kubeadm
```

安装`1.24.1-00` 版本的`kubeadm`.

```bash
sudo apt-get -y install kubelet=1.24.1-00 kubeadm=1.24.1-00 kubectl=1.24.1-00 --allow-downgrades
```

## 配置主节点

### kubeadm初始化

在承担主节点的虚拟机里配置控制平面（Control Plane）。

检查`kubeadm `当前默认配置参数。

```bash
kubeadm config print init-defaults
```

类似结果如下。保存默认配置的结果，后续会作为参考。

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: 1.24.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

模拟安装和正式安装。 

通过命令 `kubeadm init` 进行主节点的初始化，下面是这个命令主要参数的说明，特别是网络参数的三个选择。

* `--pod-network-cidr`: 
  * 指定pod使用的IP地址范围。如果指定了该参数，则Control Plane会自动讲指定的CIDR分配给每个节点。
  * IP地址段 `10.244.0.0/16` 是Flannel网络组件默认的地址范围。如果需要修改Flannel的IP地址段，需要在这里指定，且在部署Flannel时也要保持一致的IP段。
* `--apiserver-bind-port`: 
  * API服务（API Server）的端口，默认时6443。
* `--service-cidr`: 
  * 指定服务（service）的IP地址段，默认是`10.96.0.0/12`。

提示： 

* 服务VIPs（service VIPs），也称作集群IP（Cluster IP），通过参数 `--service-cidr`指定。
* podCIDR，也称为endpoint IP，通过参数 `--pod-network-cidr`指定。

有4种典型的网络问题：

* 高度耦合的容器与容器之间的通信：这可以通过Pod（podCIDR）和本地主机通信来解决。
* Pod对Pod通信（Pod-to-Pod）： 
  * 也被称为容器对容器通信（container-to-container）。
  * 在Flannel网络插件中的示例流程是：Pod --> veth对 --> cni0 --> flannel.1 --> 宿主机eth0 --> 宿主机eth0 --> flannel.1 --> cni0 --> veth对 --> Pod。
* Pod对Service通信（Pod-to-Service）：
  * 流程: Pod --> 内核 --> Service iptables --> Service --> Pod iptables --> Pod。
* 外部对Service通信（External-to-Service）： 
  * 负载均衡器: SLB --> NodePort --> Service --> Pod。

`kube-proxy` 是对iptables负责，不是网络流量（traffic）。 

- `kube-proxy`是Kubernetes集群中的一个组件，负责为Service提供代理服务，同时也是Kubernetes网络模型中的重要组成部分之一。`kube-proxy`会在每个节点上启动一个代理进程，通过监听Kubernetes API Server的Service和Endpoint的变化来维护一个本地的Service和Endpoint的缓存。当有请求到达某个Service时，`kube-proxy`会根据该Service的类型（ClusterIP、NodePort、LoadBalancer、ExternalName）和端口号，生成相应的iptables规则，将请求转发给Service所代理的后端Pod。

- iptables是Linux系统中的一个重要网络工具，可以设置IP包的过滤、转发和修改规则，可以实现网络层的防火墙、NAT等功能。在Kubernetes集群中，`kube-proxy`通过生成和更新iptables规则，来实现Service和Endpoint之间的转发和代理。具体来说，kube-proxy会为每个Service创建三条iptables规则链（nat表中的KUBE-SERVICES和KUBE-NODEPORTS链，以及filter表中的KUBE-SVC-XXXXX链），通过这些规则链，将请求转发到相应的Pod或者Service上。

- 因此，`kube-proxy`和iptables是紧密相关的两个组件，通过iptables规则来实现Service和Pod之间的转发和代理。这种实现方式具有可扩展性和高可用性，同时也提供了一种灵活的网络模型，可以方便地实现服务发现、负载均衡等功能。

```bash
sudo kubeadm init \
  --dry-run \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=192.244.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.24.1

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=192.244.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.24.1
```

### kubeconfig文件

给当前安装用户配置 `kubeconfig` 文件（当前例子是用户`vagrant`）。

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kubernetes 提供了一个命令行工具`kubectl`，用于使用 Kubernetes API 与 Kubernetes 集群的控制平面进行通信。

kubectl 控制 Kubernetes *cluster manager*（集群管理器）。

对于配置，kubectl 在 `$HOME/.kube`目录中查找一个名为 config 的文件，该文件是由 `kubeadm init`生成的文件 `/etc/kubernetes/admin.conf`的副本。

我们可以通过设置 `KUBECONFIG`环境变量或设置 `--kubeconfig flag`标志来指定其他 kubeconfig 文件。如果 `KUBECONFIG` 环境变量不存在，kubectl 将使用默认的 kubeconfig 文件 `$HOME/.kube/config`。

kubeconfig 文件中的 *context（上下文）* 元素用于将访问参数分组到一个方便的名称下。每个上下文都有三个参数：集群、命名空间和用户。默认情况下，kubectl 命令行工具使用当前上下文中的参数与集群通信。

文件`.kube/config`的例子：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <certificate string>
    server: https://<eth0 ip>:6443
  name: <cluster name>
contexts:
- context:
    cluster: <cluster name>
    namespace: <namespace name>
    user: <user name>
  name: <context user>@<context name>
current-context: <context name>
kind: Config
preferences: {}
users:
- name: <user name>
  user:
    client-certificate-data: <certificate string>
    client-key-data: <certificate string>
```

读取当前上下文：

```bash
kubectl config get-contexts
```

运行结果：

```console
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
```

## 安装Calico

参考安装指导 [End-to-end Calico installation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/) 。

安装 Calico：

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

运行结果：

```console
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created
```

验证Calico的状态。Calico的初始化过程可能需要几分钟时间完成。

```bash
kubectl get pod -n kube-system | grep calico
```

运行结果：

```console
calico-kube-controllers-555bc4b957-l8bn2   0/1     Pending    0          28s
calico-node-255pc                          0/1     Init:1/3   0          29s
calico-node-7tmnb                          0/1     Init:1/3   0          29s
calico-node-w8nvl                          0/1     Init:1/3   0          29s
```

验证网络状态。

```bash
sudo nerdctl network ls
```

运行结果：

```console
NETWORK ID      NAME               FILE
                k8s-pod-network    /etc/cni/net.d/10-calico.conflist
17f29b073143    bridge             /etc/cni/net.d/nerdctl-bridge.conflist
                host               
                none 
```

## 配置工作节点

使用 `kubeadm token` 来生成加入集群的令牌（token）和哈西值（hash value）。

```bash
kubeadm token create --print-join-command
```

命令用法：

```bash
sudo kubeadm join <your master node eth0 ip>:6443 --token <token generated by kubeadm init> --discovery-token-ca-cert-hash <hash key generated by kubeadm init>
```

运行结果：

```console
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: missing optional cgroups: blkio
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## 检查集群状态

查看 Kubernetes 集群的信息，包括集群 API Server 的地址、Kubernetes DNS 服务的地址等。

```bash
kubectl cluster-info
```

运行结果：

```console
bKubernetes control plane is running at https://11.0.1.129:6443
CoreDNS is running at https://11.0.1.129:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

列出集群中所有节点的详细信息，包括节点名称、节点 IP、节点标签、节点状态等。

```bash
kubectl get nodes -owide
```

列出 Kubernetes 集群中所有 Namespace 下的 Pod。

```
kubectl get pod -A
```

## 更新安装

### Bash自动补全

在每个节点上配置Bash自动补全功能。

参考[指导](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/)设置 `kubectl` [自动补全功能auto-completion](https://github.com/scop/bash-completion)。

```bash
apt install -y bash-completion

source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)

echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

### 别名

如果我们为 kubectl 设置一个别名，我们可以通过一些方法来扩展 shell 自动补全功能，使其能够与该别名一起使用。

一种方法是在 Bash shell 配置文件（如 .bashrc 或 .bash_profile）中设置别名，并为该别名指定 kubectl 的完整路径。例如，可以在 .bashrc 文件中添加以下内容：

```bash
alias k='path/to/kubectl'
```

然后，可以使用以下命令重新加载 .bashrc 文件：

```bash
source ~/.bashrc
```

接下来，可以使用`k`命令来代替`kubectl`命令，并在其后面添加相应的参数和选项。当使用自动补全功能时，Bash shell 会自动将`k`别名转换为 `kubectl` 的完整路径，并对其进行自动补全。

另一种方法是使用 Bash shell 内置的 complete 函数来为别名设置自动补全功能。例如，可以在 .bashrc 文件中添加以下内容：

```bash
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

这将为别名`k`设置自动补全功能，并将其与 kubectl 的自动补全函数`__start_kubectl`关联起来。这样，当用户在`k`命令后输入Tab键时，Bash shell 会自动调用`__start_kubectl`函数，并为用户提供相应的自动补全建议。

### 更新默认Context

查看当前的 context 列表：

```bash
kubectl config get-contexts 
```

这个命令会列出所有可用的 context 列表，并标记出当前正在使用的 context。

类似下面结果：

* `kubernetes-admin@kubernetes`是Context名。
* `kubernetes`是集群名。
* `kubernetes-admin`是用户名。
* 当前例子中没有指定名称空间。

```console
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin 
```

更新context。例如，更新context的默认名称空间等。

```bash
# Usage:
kubectl config set-context <context name> --cluster=<cluster name> --namespace=<namespace name> --user=<user name> 

# Set default namespace
kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --namespace=default --user=kubernetes-admin
```

在不同的context之间切换。

```bash
# Usage:
kubectl config use-context <context name>

# Switch to new context
kubectl config use-context kubernetes-admin@kubernetes
```

参考资料：
    * [kubectl](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
    * [commandline](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

## 安装Helm

Helm 是 Kubernetes 的包管理工具，它不随 Kubernetes 一起提供。

Helm 有三个核心概念：

- Chart（图表）是 Helm 的软件包，它包含了在 Kubernetes 集群中运行应用程序、工具或服务所需的所有资源定义。可以将其视为 Kubernetes 的 Homebrew 公式、Apt dpkg 或 Yum RPM 文件等等。
- Repository（仓库）是图表可以被收集和共享的地方，类似于 Perl 的 CPAN 存储库或 Fedora 的软件包数据库，但用于 Kubernetes 软件包。
- Release（发布）是在 Kubernetes 集群中运行的图表实例。一个图表通常可以在同一集群中安装多次，并且每次安装都会创建一个新的发布。以 MySQL 图表为例，如果想要在集群中运行两个数据库，则可以安装该图表两次，每次安装都会创建一个新的发布，每个发布都有自己的发布名称。

参考文档：

* [installation guide](https://helm.sh/docs/intro/install/)
* [binary release](https://github.com/helm/helm/releases)
* [source code](https://github.com/helm/helm).

Helm客户端安装： 

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

运行结果：

```console
Downloading https://get.helm.sh/helm-v3.9.1-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm
```

提示：
    * [`helm init`](https://helm.sh/docs/helm/helm_init/) 在Helm 3中已取消，且Tiller也一同取消。今后在集群中使用Helm时不再需要安装Tiller。
    * `helm search`可以用来搜索两种不同类型的资源：
        * `helm search hub`在[Artifact Hub](https://artifacthub.io/)中搜索，这个hub里列出来自数十个不同仓库的 Helm Chart。
        * `helm search repo` 命令用于搜索已添加到本地 Helm 客户端的仓库（使用 `helm repo add` 命令）。此搜索是在本地数据上进行的，不需要公共网络连接。

参考资料：[Helming development](../foundamentals/helming.md)

## 重置集群

注意：下面的操作会重置当前集群（删除集群）。

删除集群中所有节点。

```bash
kubeadm reset
```

清除`iptables`中已定义的规则。

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

清除`IPVS`中定义的规则（如果使用`IPVS`）。

```bash
ipvsadm --clear
```
