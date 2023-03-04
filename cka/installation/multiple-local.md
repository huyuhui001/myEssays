# 多节点虚拟机安装Kubernetes

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

在所有节点上安装Kubernetes。

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

坚持当前`kubeadm`的版本。

```bash
apt policy kubeadm
```

安装`1.24.1-00` 版本的`kubeadm`.

```bash
sudo apt-get -y install kubelet=1.24.1-00 kubeadm=1.24.1-00 kubectl=1.24.1-00 --allow-downgrades
```

## 配置主节点

### kubeadm init

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

### kubeconfig file

Set `kubeconfig` file for current user (here it's `vagrant`).

```console
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kubernetes provides a command line tool `kubectl` for communicating with a Kubernetes cluster's control plane, using the Kubernetes API.

kubectl controls the Kubernetes *cluster manager*.

For configuration, kubectl looks for a file named config in the `$HOME/.kube` directory, which is a copy of file `/etc/kubernetes/admin.conf` generated by `kubeadm init`. 

We can specify other kubeconfig files by setting the `KUBECONFIG` environment variable or by setting the `--kubeconfig flag`.  If the `KUBECONFIG` environment variable doesn't exist, kubectl uses the default kubeconfig file, `$HOME/.kube/config`.

A *context* element in a kubeconfig file is used to group access parameters under a convenient name. Each context has three parameters: cluster, namespace, and user. By default, the kubectl command-line tool uses parameters from the current context to communicate with the cluster.

A sample of `.kube/config`.

```
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

To get the current context:

```console
kubectl config get-contexts
```

Result

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
```

## Install Calico

Here is guidance of [End-to-end Calico installation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/).
Detail practice demo, can be found in section "Install Calico" of "A1.Discussion" below. 

Install Calico

```console
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Result

```
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

Verify status of Calico. It may take minutes to complete initialization.

```console
kubectl get pod -n kube-system | grep calico
```

Result

```
calico-kube-controllers-555bc4b957-l8bn2   0/1     Pending    0          28s
calico-node-255pc                          0/1     Init:1/3   0          29s
calico-node-7tmnb                          0/1     Init:1/3   0          29s
calico-node-w8nvl                          0/1     Init:1/3   0          29s
```

Verify network status.

```console
sudo nerdctl network ls
```

Result

```
NETWORK ID      NAME               FILE
                k8s-pod-network    /etc/cni/net.d/10-calico.conflist
17f29b073143    bridge             /etc/cni/net.d/nerdctl-bridge.conflist
                host               
                none 
```

## Setup Work Nodes

Use `kubeadm token` to generate the join token and hash value.

```console
kubeadm token create --print-join-command
```

Command usage:

```console
sudo kubeadm join <your master node eth0 ip>:6443 --token <token generated by kubeadm init> --discovery-token-ca-cert-hash <hash key generated by kubeadm init>
```

Result looks like below.

```
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

## Check Cluster Status

Cluster info:

```console
kubectl cluster-info
```

Output

```
Kubernetes control plane is running at https://11.0.1.129:6443
CoreDNS is running at https://11.0.1.129:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Node info:

```
kubectl get nodes -owide
```

Pod info:

```
kubectl get pod -A
```

## Post Installation

### Bash Autocomplete

On each node.

Set `kubectl` [auto-completion](https://github.com/scop/bash-completion) following the [guideline](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/).

```
apt install -y bash-completion

source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)

echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

### Alias

If we set an alias for kubectl, we can extend shell completion to work with that alias:

```
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

### Update Default Context

Get current context.

```console
kubectl config get-contexts 
```

In below result, we know:

* Contenxt name is `kubernetes-admin@kubernetes`.
* Cluster name is `kubernetes`.
* User is `kubernetes-admin`.
* No namespace explicitly defined.

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin 
```

To set a context with new update, e.g, update default namespace, etc.. 

```console
# Usage:
kubectl config set-context <context name> --cluster=<cluster name> --namespace=<namespace name> --user=<user name> 

# Set default namespace
kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --namespace=default --user=kubernetes-admin
```

To switch to a new context.

```console
kubectl config use-context <context name>
```

```console
kubectl config use-context kubernetes-admin@kubernetes
```

!!! Reference
    * [kubectl](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
    * [commandline](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

## Install Helm

Helm is the Kubernetes package manager. It doesn't come with Kubernetes. 

Three concepts of helm:

* A *Chart* is a Helm package. 
  * It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster. 
  * Think of it like the Kubernetes equivalent of a Homebrew formula, an Apt dpkg, or a Yum RPM file.
* A *Repository* is the place where charts can be collected and shared. 
  * It's like Perl's CPAN archive or the Fedora Package Database, but for Kubernetes packages.
* A *Release* is an instance of a chart running in a Kubernetes cluster. 
  * One chart can often be installed many times into the same cluster. And each time it is installed, a new release is created. 
  * Consider a MySQL chart. If you want two databases running in your cluster, you can install that chart twice. Each one will have its own release, which will in turn have its own release name.

Reference:

* [installation guide](https://helm.sh/docs/intro/install/)
* [binary release](https://github.com/helm/helm/releases)
* [source code](https://github.com/helm/helm).

Helm Client Installation: 

```console
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Output:

```
Downloading https://get.helm.sh/helm-v3.9.1-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm
```

!!! Note
    * [`helm init`](https://helm.sh/docs/helm/helm_init/) does not exist in Helm 3, following the removal of Tiller. You no longer need to install Tiller in your cluster in order to use Helm.
    * `helm search` can be used to search two different types of source:
        * `helm search hub` searches the [Artifact Hub](https://artifacthub.io/), which lists helm charts from dozens of different repositories.
        * `helm search repo` searches the repositories that you have added to your local helm client (with helm repo add). This search is done over local data, and no public network connection is needed.

!!! Reference
    [Helming development](../foundamentals/helming.md)

## Reset cluster

!!! Caution
    Below steps will destroy current cluster. 

Delete all nodes in the cluster.

```console
kubeadm reset
```

Clean up rule of `iptables`.

```console
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

Clean up rule of `IPVS` if using `IPVS`.

```console
ipvsadm --clear
```
