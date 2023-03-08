# # CKA自学笔记3:阿里云ECS安装Kubernetes

## 摘要

在阿里云ECS装三台Ubuntu虚拟机。在Ubuntu虚拟机中安装基于Containerd的Kubernetes系统，并分别配置一个主节点Master和两个工作节点Worker，。

## 准备工作

注册阿里云账号： [Alibaba Cloud home console](https://home.console.aliyun.com/home/dashboard/ProductAndService)。注意保留访问密钥key文件，只能导出一次，当前练习中key文件是`aliyun-root`。

参考下面配置注册申请三个ECS（Elastic Computer Service）服务实例：

* 主机：2vCPU+4GiB
* 操作系统：Ubuntu  20.04 x86_64
* 实例类型：ecs.sn1.medium 
* 实例名称：cka001, cka002, cka003
* 网络配置：both public IPs and private IPs
* 最大网络带宽：100Mbps (Peak Value)
* 云盘：40GiB
* 支付方式：抢占式实例

在本地打开终端窗口，通过密钥文件`aliyun-root`访问远程ECS节点 `cka001` 。

```bash
ssh -i aliyun-root root@cka001
```

创建一个普通用户，用来安装Kubernetes，当前练习中创建用户 `vagrant`，且修改该用户的主要组为 `sudo` 次要组包含 `root`。

```bash
adduser vagrant
usermod -g sudo vagrant
usermod -a -G root vagrant
```

新开一个本地终端窗口，为用户`vagrant`创建密钥key。

```bash
# Windows
ssh-keygen.exe

# Linux
ssh-keygen
```

上面的命令会生成2个文件，当前练习中这2个文件是 `aliyun-vagrant` and `aliyun-vagrant.pub`

通过`sftp`命令将公钥文件 `aliyun-vagrant.pub` 上传到远程节点 `cka001`。

```bash
sftp -i aliyun-root root@cka001
put aliyun-vagrant.pub
```

新开一个终端窗口，用`root`的密钥登录`cka001`节点。 
将上一步上传的密钥文件`aliyun-vagrant.pub` 从`/root`目录拷贝到 `/home/vagrant/.ssh/`。
将公钥文件 `aliyun-vagrant.pub` 重命名为 `authorized_keys`。
更改文件`authorized_keys` 的所有者owner为 `vagrant`.
更改文件`authorized_keys` 的主要组为 `sudo`。

```bash
mkdir /home/vagrant/.ssh/
mv aliyun-james.pub /home/vagrant/.ssh/authorized_keys
chown vagrant.sudo /home/vagrant/.ssh/authorized_keys
chmod 600 /home/vagrant/.ssh/authorized_keys
```

检查文件 `/etc/ssh/sshd_config`，确定密码登录验证参数`asswordAuthentication`设定为`no`，即只能通过证书远程登录。 

```bash
cat /etc/ssh/sshd_config
```

新开一个终端窗口，使用用户vagrant登录远程节点`cka001`，验证用户`vagrant`能通过前面创建的证书登录节点`cka001`。

```bash
ssh -i aliyun-vagrant vagrant@cka001
```

重复上述步骤，通过`sftp`命令将公钥文件 `aliyun-vagrant.pub`分别上传到远程节点 `cka002` 和 `cka003`，且完成同样的配置，使用户`vagrant`也能通过密钥文件登录远程节点 `cka002` 和 `cka003`。

至此，用户 `vagrant` 可以通过密钥文件`aliyun-vagrant`从本地终端窗口登录远程节点 `cka001`, `cka002` 和`cka003` 。

下面所有步骤都是通过用户 `vagrant`完成。

## 初始化ECS节点

### 配置文件/etc/hosts

更新所有ECS节点的文件/etc/hosts，添加其他节点的私有IP（private IP）。

```bash
vi /etc/hosts
```

### 禁用firewall

在所有节点上禁用防火墙。

```bash
sudo ufw disable
```

检查防火墙状态。

```bash
sudo ufw status verbose
```

## 关闭swap

在所有节点上关闭swap。

```bash
sudo swapoff -a
```

## 设置时区和地域

在所有节点设定时区和地域。这一布在初始化ECS时候已经完成。可以通过下面命令进行设定。

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
sudo echo 'LANG="en_US.UTF-8"' >> /etc/profile
source /etc/profile
```

通过下面命令检查时区和地域的设置。

```bash
ll /etc/localtime
```

```console
lrwxrwxrwx 1 root root 33 Jul  5 14:51 /etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai
```

## 内核设置

在所有节点上执行下面的命令以配置内核。

使用模块overlay：

创建Containerd服务配置文件 `/etc/modules-load.d/containerd.conf` ，如果已存在则跳过这一步。配置这个文件的目的是为了加载模块 `overlay` 和 `br_netfilter`到内核中。

服务Containerd依赖模块`overlay` 实现[overlay-filesystem](https://developer.aliyun.com/article/660712)文件系统功能。

Linux中的overlay模块提供了创建两个目录的合并视图的能力，这两个目录称为层。它经常被用于实现联合挂载，这是一种将两个或更多目录一起挂载的方式，就像它们是一个目录一样（union-filesystems）。

overlay模块在容器技术中被广泛使用，比如Docker，因为它允许多个容器共享基础镜像，同时保持它们自己的文件系统。

要使用overlay模块，需要两个目录：较低的目录（lower directory）和较高的目录（upper directory）。较低的目录通常是只读的，包含原始文件，而较高的目录是可读写的，包含对文件的更改。当请求文件时，overlay模块首先查找上层目录，如果未找到，则查找下层目录。

比如：

创建两个目录，一个用于较低的目录，一个用于较高的目录。然后使用overlay文件系统类型将它们挂载起来：

```bash
sudo mkdir /lower sudo mkdir /upper sudo mount -t overlay overlay -o lowerdir=/lower,upperdir=/upper /merged
```

在上面的例子中，两个目录的合并视图被创建在`/merged`目录中。在`/merged`目录中对文件所做的任何更改都存储在上层目录中，而原始文件仍然在下层目录中。

使用模块br_netfilter：

`br_netfilter`是Linux内核中的一个模块，它提供了一种机制来过滤网桥的网络流量。该模块允许管理员配置规则，以允许或拒绝特定的网络流量通过网桥。

网桥是一种网络设备，它可以连接多个网络段，并转发流量以使不同的网络段之间通信。`br_netfilter`模块可以用来限制或过滤这些流量。

当启用了`br_netfilter`模块时，它会自动启用一个称为`bridge-nf`的功能，该功能将在网络流量通过网桥时应用规则。管理员可以使用iptables等工具来配置这些规则。例如，我们可以设定允许从一个网络段到另一个网络段的流量，或者拒绝来自特定IP地址或端口的流量。

在Kubernetes中，`br_netfilter`模块主要用于启用Kubernetes服务的流量转发和负载均衡。这些服务使用了Linux内核中的iptables规则来管理流量，这些规则是通过`br_netfilter`模块实现的。

具体来说，当我们在Kubernetes集群中创建一个服务时，该服务将分配一个虚拟IP地址，用于代表服务。然后，通过iptables规则，将这个虚拟IP地址映射到一个或多个后端Pod的IP地址，以便在需要时将流量路由到这些Pod。

在这个过程中，`br_netfilter`模块负责监视服务的流量，并根据iptables规则进行转发和负载均衡。这包括过滤来自不受信任源的流量以及限制服务的访问权限。

需要注意的是，为了启用Kubernetes服务的流量转发和负载均衡，`br_netfilter`模块必须在所有节点上启用，并且必须配置正确的iptables规则。

由于`br_netfilter`模块的作用非常关键，因此在升级或更改系统时需要特别注意它的配置和状态。

下面命令将模块`overlay`和`br_netfilter`添加到配置文件`containerd.conf`中。

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

手工加载模块`overlay` 和 `br_netfilter` 。

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

验证模块是否已被加载。

```bash
lsmod | grep br_netfilter
```

创建文件 `99-kubernetes-cri.conf` 用来配置Kubernetes CRI。

Kubernetes CRI (Container Runtime Interface) 是 Kubernetes 用于管理容器的插件架构。它定义了容器运行时所需的 API 和契约，使得 Kubernetes 可以与任何符合该接口标准的容器运行时交互。

CRI 最初于 Kubernetes 1.5 版本中引入，作为将容器运行时从 kubelet 中解耦出来的一种方法。在 Kubernetes 中，kubelet 是运行在节点上的主要代理程序，它负责在节点上运行容器并与 Kubernetes API 交互。通过引入 CRI，kubelet 可以使用不同的容器运行时（如 Docker、CRI-O、containerd 等）来运行容器，而无需了解容器运行时的内部细节。这种解耦使得 Kubernetes 更加灵活和可扩展，并使容器运行时的维护更加简单。

总之，Kubernetes CRI 定义了 Kubernetes 与容器运行时之间的接口，使得 Kubernetes 可以使用不同的容器运行时来管理容器，并使 Kubernetes 更加灵活和可扩展。

设置参数 `net/bridge/bridge-nf-call-iptables=1` ，使 Linux 桥接模块上运行的容器能够通过 iptables 进行网络包过滤（filtering）和转发（forwarding）。

这个参数的作用是让 Linux 网络桥接能够支持 iptables 的 NAT(Network Address Translation)和过滤功能。具体来说，当容器网络流量通过 Linux 网桥桥接到宿主机网络时，如果这个参数为 1，则 iptables 规则将被应用到容器的网络流量上。如果设置为 0，则不会应用 iptables 规则，这可能会导致容器网络的不稳定性。

参考[Why `net/bridge/bridge-nf-call-iptables=1` need to be enable by Kubernetes](https://cloud.tencent.com/developer/article/1828060)）。

IP转发（IP forwarding）也被称为路由（routing）。在Linux中，它也被称为内核IP转发，因为它使用内核变量`net.ipv4.ip_forward`来启用或禁用IP转发功能。默认值为`ip_forward=0`。因此，Linux的IP转发功能默认情况下是被禁用的。

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

命令`sysctl` 从目录 `/proc/sys` 读取信息。`/proc/sys`是一个虚拟目录，其中包含可用于查看和设置当前内核参数的文件对象。

通过`sysctl -w net.ipv4.ip_forward=1`命令，更改立即生效，但不是永久的。在系统重启后，将加载默认值。要永久设置参数，需要将设置写入`/etc/sysctl.conf`或`/etc/sysctl.d`目录中的另一个配置文件：

```bash
sudo sysctl --system
```

验证参数是否生效。

```bash
sysctl net.ipv4.ip_forward
```

## 安装Containerd

在所有节点上安装Containerd服务。参考资料：[containerd for Ops and Admins](https://github.com/containerd/containerd/blob/main/docs/ops.md)

安装前备份Ubuntu安装源配置文件。

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

添加合适的安装源。基于阿里ECS的Ubuntu 20.04，已经预配置了阿里内部的源，这一步只需要检查一下是否阿里源已配置。

```bash
cat > /etc/apt/sources.list << EOF
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal main restricted
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal main restricted
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-updates main restricted
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-updates main restricted
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal universe
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal universe
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-updates universe
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-updates universe
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal multiverse
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal multiverse
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-updates multiverse
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-updates multiverse
deb http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu/ focal-backports main restricted universe multivers
deb http://mirrors.cloud.aliyuncs.com/ubuntu focal-security main restricted
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu focal-security main restricted
deb http://mirrors.cloud.aliyuncs.com/ubuntu focal-security universe
deb-src http://mirrors.cloud.aliyuncs.com/ubuntu focal-security universe
# deb http://mirrors.cloud.aliyuncs.com/ubuntu focal-security multiverse
# deb-src http://mirrors.cloud.aliyuncs.com/ubuntu focal-security multiverse
EOF
```

安装Containered。

```bash
sudo apt-get update && sudo apt-get install -y containerd
```

配置Containerd。修改文件 `/etc/containerd/config.toml`。

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

修改参数`sandbox_image` 的值为`"registry.aliyuncs.com/google_containers/pause:3.6"`。
修改参数`SystemdCgroup `的值为`true`。

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

重启Containerd服务。

```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

## 安装nerdctl

在所有节点上安装nerdctl服务。

[`nerdctl`](https://github.com/containerd/nerdctl) 服务支持Contanerd所提供的容器化特性，特别是一些Docker不具备的新特性。

二进制安装包可以通过这个链接取得: [Releases · containerd/nerdctl · GitHub](https://github.com/containerd/nerdctl/releases) 。

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
nerdctl -n k8s.io ps
```

## 安装kubeadm

在所有节点上安装Kubeadm，kubectl，kubelet。

安装和升级Ubuntu系统依赖包 `apt-transport-https`,  `ca-certificates`, `curl`。

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
```

安装gpg证书。

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
```

添加Kubernetes安装源。

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ \
  kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

安装和升级Ubuntu系统依赖包。

```bash
sudo apt-get update
sudo apt-get install ebtables
sudo apt-get install libxtables12
sudo apt-get upgrade iptables
```

检查当前可用的`kubeadm`版本。

```bash
apt policy kubeadm
```

当前安装`1.24.0-00` 版本的`kubeadm`，后续会升级到 `1.24.2` 版本。

```bash
sudo apt-get -y install kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00 --allow-downgrades
```

## 配置主节点

### kubeadm初始化

在承担主节点的虚拟机里配置控制平面（Control Plane）。

检查`kubeadm` 当前默认配置参数。

```bash
kubeadm config print init-defaults
```

类似结果如下。保存默认配置的结果，后续会作为参考。

```
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

- `--pod-network-cidr`:
  - 指定pod使用的IP地址范围。如果指定了该参数，则Control Plane会自动讲指定的CIDR分配给每个节点。
  - IP地址段 `10.244.0.0/16` 是Flannel网络组件默认的地址范围。如果需要修改Flannel的IP地址段，需要在这里指定，且在部署Flannel时也要保持一致的IP段。
- `--apiserver-bind-port`:
  - API服务（API Server）的端口，默认时6443。
- `--service-cidr`:
  - 指定服务（service）的IP地址段，默认是`10.96.0.0/12`。

提示：

- 服务VIPs（service VIPs），也称作集群IP（Cluster IP），通过参数 `--service-cidr`指定。
- podCIDR，也称为endpoint IP，通过参数 `--pod-network-cidr`指定。

有4种典型的网络问题：

- 高度耦合的容器与容器之间的通信：这可以通过Pod（podCIDR）和本地主机通信来解决。
- Pod对Pod通信（Pod-to-Pod）：
  - 也被称为容器对容器通信（container-to-container）。
  - 在Flannel网络插件中的示例流程是：Pod --> veth对 --> cni0 --> flannel.1 --> 宿主机eth0 --> 宿主机eth0 --> flannel.1 --> cni0 --> veth对 --> Pod。
- Pod对Service通信（Pod-to-Service）：
  - 流程: Pod --> 内核 --> Service iptables --> Service --> Pod iptables --> Pod。
- 外部对Service通信（External-to-Service）：
  - 负载均衡器: SLB --> NodePort --> Service --> Pod。

`kube-proxy` 是对iptables负责，不是网络流量（traffic）。

- `kube-proxy`是Kubernetes集群中的一个组件，负责为Service提供代理服务，同时也是Kubernetes网络模型中的重要组成部分之一。`kube-proxy`会在每个节点上启动一个代理进程，通过监听Kubernetes API Server的Service和Endpoint的变化来维护一个本地的Service和Endpoint的缓存。当有请求到达某个Service时，`kube-proxy`会根据该Service的类型（ClusterIP、NodePort、LoadBalancer、ExternalName）和端口号，生成相应的iptables规则，将请求转发给Service所代理的后端Pod。

- iptables是Linux系统中的一个重要网络工具，可以设置IP包的过滤、转发和修改规则，可以实现网络层的防火墙、NAT等功能。在Kubernetes集群中，`kube-proxy`通过生成和更新iptables规则，来实现Service和Endpoint之间的转发和代理。具体来说，kube-proxy会为每个Service创建三条iptables规则链（nat表中的KUBE-SERVICES和KUBE-NODEPORTS链，以及filter表中的KUBE-SVC-XXXXX链），通过这些规则链，将请求转发到相应的Pod或者Service上。

- 因此，`kube-proxy`和iptables是紧密相关的两个组件，通过iptables规则来实现Service和Pod之间的转发和代理。这种实现方式具有可扩展性和高可用性，同时也提供了一种灵活的网络模型，可以方便地实现服务发现、负载均衡等功能。

```bash
sudo kubeadm init \
  --dry-run \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr 11.244.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.24.0

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr 11.244.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.24.0

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr 11.244.0.0/16 \
  --kubernetes-version=v1.24.0
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

```
kubectl config get-contexts
```

运行结果：

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
```

## 配置工作节点

使用 `kubeadm token` 来生成加入集群的令牌（token）和哈西值（hash value）。

```bash
kubeadm token create --print-join-command
```

在所有工作节点上执行下面的命令，将工作节点加入Kubernetes集群。

```bash
# kubeadm join <your master node eth0 ip>:6443 --token <token generated by kubeadm init> --discovery-token-ca-cert-hash <hash key generated by kubeadm init>
```

执行下面命令检查所有节点的状态。 当前所有节点的状态都是 `NotReady`。目前不需要做什么，后面我们会安装相关的网络服务（Calico 或 Flannel），各节点的状态就会变成Ready状态。

## 安装Calico或Flannel

在控制平面Control Plane上安装Calico或者Flannel。如果需要配置网络策略，则选择Calico。 

### 安装Flannel

[Flannel](https://github.com/flannel-io/flannel) 是为 Kubernetes 设计的一种简单易用的配置三层网络的方法。

部署Flannel：

在 `kube-flannel.yml` 中，我们可以获取 Flannel 的默认网络设置，它与我们在使用 `kubeadm` 初始化集群时指定的参数 `--pod-network-cidr=10.244.0.0/16` 相同。

```json
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

创建Flannel服务。

```bash
apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

输出结果：

```console
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

### 安装Calico

安装指导手册：[End-to-end Calico installation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/)。

下载并安装Calico服务。

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

输出结果：

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

验证Calico服务状态：

```bash
kubectl get pod -n kube-system | grep calico
```

输出结果：

```bash
calico-kube-controllers-555bc4b957-l8bn2   0/1     Pending    0          28s
calico-node-255pc                          0/1     Init:1/3   0          29s
calico-node-7tmnb                          0/1     Init:1/3   0          29s
calico-node-w8nvl                          0/1     Init:1/3   0          29s
```

检查集群的网络状态：

```bash
sudo nerdctl network ls
```

输出结果：

```console
NETWORK ID    NAME               FILE
              k8s-pod-network    /etc/cni/net.d/10-calico.conflist
0             bridge             /etc/cni/net.d/nerdctl-bridge.conflist
              host               
              none 
```

## 检查集群状态

在主节点上执行命令`kubectl cluster-info` 可以得到下面的信息：

* 控制平面（control plane）运行在 `https://<mster node ip>:6443`
* CoreDNS服务运行在 `https://<mster node ip>:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy`

```bash
kubectl cluster-info
```

查看节点运行状态。此时，所有节点都是`Ready`的正常状态了。

- OS Image: Ubuntu 20.04.4 LTS
- Kernel Version: 5.4.0-122-generic
- Container Runtime: containerd://1.5.9

```bash
kubectl get nodes -owide
```

输出结果：

```console
NAME     STATUS   ROLES           AGE     VERSION
cka001   Ready    control-plane   13m     v1.24.0
cka002   Ready    <none>          8m35s   v1.24.0
cka003   Ready    <none>          8m26s   v1.24.0
```

查看Pods的状态。

```bash
kubectl get pod -A
```

输出结果：

```console
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-555bc4b957-l8bn2   1/1     Running   0          7m18s
kube-system   calico-node-255pc                          1/1     Running   0          7m19s
kube-system   calico-node-7tmnb                          1/1     Running   0          7m19s
kube-system   calico-node-w8nvl                          1/1     Running   0          7m19s
kube-system   coredns-74586cf9b6-4jwmk                   1/1     Running   0          15m
kube-system   coredns-74586cf9b6-c5mll                   1/1     Running   0          15m
kube-system   etcd-cka001                                1/1     Running   0          15m
kube-system   kube-apiserver-cka001                      1/1     Running   0          15m
kube-system   kube-controller-manager-cka001             1/1     Running   0          15m
kube-system   kube-proxy-dmj2t                           1/1     Running   0          15m
kube-system   kube-proxy-n77zw                           1/1     Running   0          11m
kube-system   kube-proxy-qs6rf                           1/1     Running   0          11m
kube-system   kube-scheduler-cka001                      1/1     Running   0          15m
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

- `kubernetes-admin@kubernetes`是Context名。
- `kubernetes`是集群名。
- `kubernetes-admin`是用户名。
- 当前例子中没有指定名称空间。

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

- [kubectl](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [commandline

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

## 排错

### 错误1：

报错信息：

The connection to the server <master>:6443 was refused - did you specify the right host or port?

解决尝试：

[Reference](https://discuss.kubernetes.io/t/the-connection-to-the-server-host-6443-was-refused-did-you-specify-the-right-host-or-port/552/15)

检查文件kubeconfig的内容和文件路径是否正确。 

检查环境变量设置。

```bash
env | grep -i kub
```

检查容器运行状态。

```bash
sudo systemctl status containerd.service 
```

检查kubelet服务。

```bash
sudo systemctl status kubelet.service 
```

检查`6443`端口监听状态。

```bash
netstat -pnlt | grep 6443
```

检查防火墙状态。

```bash
sudo systemctl status firewalld.service
```

检查kubelet日志。

```
journalctl -xeu kubelet
```

### 错误2：

报错信息：

"Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"

尝试方法：

重启Containerd服务。

```
sudo systemctl restart containerd
sudo systemctl status containerd
```

## 
