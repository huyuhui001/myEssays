# Multiple Nodes Installation

## Local VM setting

VMWare Setting.

* VMnet1: host-only, subnet: 192.168.150.0/24
* VMnet8: NAT, subnet: 11.0.1.0/24

Create guest machine with VMWare Player.

* 4 GB RAM
* 1 CPUs with 2 Cores
* Ubuntu Server 22.04
* NAT

!!! info
    Kubernetes running on Containerd.


## Ubuntu Post Installation

!!! info
    Log onto each VM with the account created during Ubuntu installation, and perform below tasks on every VM. 


Create user `vagrant` on all guests.
```console
sudo adduser vagrant
sudo usermod -aG adm,sudo,syslog,cdrom,dip,plugdev,lxd,root vagrant
sudo passwd vagrant
```

Set password for `root` on all guests.
```console
sudo passwd root
```

Enable root ssh logon.
```console
sudo vi /etc/ssh/sshd_config
```

Update parameter `PermitRootLogin` from `prohibit-password` to `yes`.
```
PermitRootLogin yes
#PermitRootLogin prohibit-password
```

Restart the sshd service.
```console
sudo systemctl restart sshd
```

Change host name, e.g., `ubu1`.
```console
sudo hostnamectl set-hostname ubu1
sudo hostnamectl set-hostname ubu1 --pretty
```

Verify if the hostname is set to expected name, e.g., `ubu1`.
```console
cat /etc/machine-info
```

Verify if the hostname is set to expected name, e.g., `ubu1`.
```console
cat /etc/hostname
```

Verify if the hostname of `127.0.1.1` is set to expected name, e.g., `ubu1`. And add all nodes into the file `/etc/hosts`.
```console
sudo vi /etc/hosts
```
Related setting looks like below.
```
127.0.1.1 ubu1
11.0.1.129 ubu1
11.0.1.130 ubu2
11.0.1.131 ubu3
11.0.1.132 ubu4
```

Create file `/etc/netplan/00-installer-config.yaml`.
```console
sudo vi /etc/netplan/00-installer-config.yaml
```

Update it with information below to set VM with fixed IP with actual IP address, e.g, `11.0.1.129`.
```
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

Effect above change.
```console
sudo netplan apply
```

!!! Attention
    The current ssh connection will be broken due to network setting change.


Disable swap and firewall on all nodes.
```console
sudo swapoff -a
sudo ufw disable
sudo ufw status verbose
```
And comment the last line of swap setting in file `/etc/fstab`. Need *reboot* guest here.
```console
sudo vi /etc/fstab
```
Result likes below.
```
/dev/disk/by-uuid/df370d2a-83e5-4895-8c7f-633f2545e3fe / ext4 defaults 0 1
# /swap.img     none    swap    sw      0       0
```

Setup timezone on all nodes
```console
sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

sudo cp /etc/profile /etc/profile.bak
echo 'LANG="en_US.UTF-8"' | sudo tee -a /etc/profile

source /etc/profile
```

Something like this after execute command `ll /etc/localtime`
```
lrwxrwxrwx 1 root root 33 Jul 15 22:00 /etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai
```

Kernel setting.
```console
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Load to kernel.
```console
sudo modprobe overlay
sudo modprobe br_netfilter
```

Network setting.
```console
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Effect changes above.
```console
sudo sysctl --system
```

!!! Attention
    Reboot the VM.


!!! Attention
    Log onto the VM with account `vagrant` to verify if above changes were updated as expected.

    * IP address.
    ```console
    ip addr list
    ```
    * Hostname.
    ```console
    cat /etc/machine-info
    cat /etc/hostname
    hostname
    ```
    * Firewall.
    ```console
    sudo ufw status verbose
    ```
    * Kernel setting.
    ```console
    lsmod | grep -i overlay
    lsmod | grep -i br_netfilter
    ```
    * Network setting.
    ```console
    sudo sysctl -a | grep -i net.bridge.bridge-nf-call-ip*
    sudo sysctl -a | grep -i net.ipv4.ip_forward
    ```


## Install Containerd

Install Containerd sevice on all nodes.

Backup source file.
```console
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

Install Containered.
```console
sudo apt-get update && sudo apt-get install -y containerd
```

Configure Containerd. Modify file `/etc/containerd/config.toml`.
```console
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

Update `sandbox_image` with new value `"registry.aliyuncs.com/google_containers/pause:3.6"`.
Update `SystemdCgroup ` with new value `true`.
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

Restart Containerd service.
```console
sudo systemctl restart containerd
sudo systemctl status containerd
```

  
## Install nerdctl

Install nerdctl sevice on all nodes.

The goal of [`nerdctl`](https://github.com/containerd/nerdctl) is to facilitate experimenting the cutting-edge features of containerd that are not present in Docker.

Binaries are available here: https://github.com/containerd/nerdctl/releases

```console
wget https://github.com/containerd/nerdctl/releases/download/v0.22.2/nerdctl-0.22.2-linux-amd64.tar.gz
tar -zxvf nerdctl-0.22.2-linux-amd64.tar.gz
sudo cp nerdctl /usr/bin/
```

Verify nerdctl.
```console
sudo nerdctl --help
```

To list local Kubernetes containers.
```console
sudo nerdctl -n k8s.io ps
```



## Install Kubernetes

Install Kubernetes on all nodes.

Install dependencied packages.
```console
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
```

Install gpg certificate.
```console
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
```
Add Kubernetes repo. 
```console
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update and install dependencied packages.
```console
sudo apt-get update
sudo apt-get install ebtables
sudo apt-get install libxtables12
sudo apt-get upgrade iptables
```

Check available versions of kubeadm.
```console
apt policy kubeadm
```

Install `1.24.1-00` version.
```console
sudo apt-get -y install kubelet=1.24.1-00 kubeadm=1.24.1-00 kubectl=1.24.1-00 --allow-downgrades
```


## Setup Master Node

### kubeadm init

Set up Control Plane on VM playing master node.

Check kubeadm default parameters for initialization.
```console
kubeadm config print init-defaults
```
Reuslt:
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

Dry rune and run. Save the output, which will be used later on work nodes.

With `kubeadm init` to initiate cluster, we need understand below three options about network.

* `--pod-network-cidr`: 
    * Specify range of IP addresses for the pod network. If set, the control plane will automatically allocate CIDRs for every node.
    * Be noted that `10.244.0.0/16` is default range of flannel. If it's changed here, please do change the same when deploy `Flannel`.
* `--apiserver-bind-port`: 
    * Port for the API Server to bind to. (default 6443)
* `--service-cidr`: 
    * Use alternative range of IP address for service VIPs. (default "10.96.0.0/12")

Note: 

* service VIPs (a.k.a. Cluster IP), specified by option `--service-cidr`.
* podCIDR (a.k.a. endpoint IP)，specified by option `--pod-network-cidr`.

There are 4 distinct networking problems to address:

* Highly-coupled container-to-container communications: this is solved by Pods (podCIDR) and localhost communications.
* Pod-to-Pod communications: 
    * a.k.a. container-to-container. 
    * Example with Flannel, the flow is: Pod --> veth pair --> cni0 --> flannel.1 --> host eth0 --> host eth0 --> flannel.1 --> cni0 --> veth pair --> Pod.
* Pod-to-Service communications:
    * Flow: Pod --> Kernel --> Servive iptables --> service --> Pod iptables --> Pod
* External-to-Service communications: 
    * LoadBalancer: SLB --> NodePort --> Service --> Pod

`kube-proxy` is responsible for iptables, not traffic. 

```console
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
