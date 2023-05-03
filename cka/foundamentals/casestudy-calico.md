# 主题讨论:安装Calico

演示场景：安装Calico

这是一个关于如何配置和测试Calico网络的简要步骤：

* Calico数据库（Datastore）：Calico支持使用etcd或Kubernetes API server作为数据存储后端。选择并部署其中一个数据存储后端。
* 配置IP池：为了为Kubernetes集群中的节点分配IP地址，需要配置IP池。可以通过Calico自定义资源（CRD）来定义IP池。
* 安装CNI插件：CNI插件负责在节点上创建和删除网络接口，它们是应用程序容器和物理网络之间的桥梁。需要在Kubernetes节点上安装Calico CNI插件。
* 安装Typha：Typha是Calico中央控制平面的一个组件。它从Kubernetes API server中获取网络策略和其他信息，并将它们分发给所有节点上的calico/node。
* 安装calico/node：calico/node是一个运行在Kubernetes节点上的守护进程。它管理节点上的网络接口，并为容器分配和释放IP地址。
* 测试网络：在完成上述步骤后，可以通过在Pod之间进行网络通信来测试Calico网络是否正常工作。可以创建两个运行在不同节点上的Pod，并尝试从一个Pod ping另一个Pod。如果ping成功，则表示Calico网络已成功配置和运行。

## Calico数据库

为了将Kubernetes用作Calico数据存储库，我们需要定义Calico使用的自定义资源。

下载并检查Calico自定义资源定义列表，并在文件编辑器中打开它。

```bash
wget https://projectcalico.docs.tigera.io/manifests/crds.yaml
```

在 Kubernetes 中创建 Calico 的自定义资源。

```bash
kubectl apply -f crds.yaml
```

安装`calicoctl`。

下载 `calicoctl` 二进制文件到一个可以访问 Kubernetes 的 Linux 主机上，以直接与 Calico 数据存储交互。

最新版的calicoctl可以通过[git page](https://github.com/projectcalico/calico/releases)进行下载，需要用实际版本号替换下面的`v3.23.2`的版本号。

```bash
wget https://github.com/projectcalico/calico/releases/download/v3.23.3/calicoctl-linux-amd64
chmod +x calicoctl-linux-amd64
sudo cp calicoctl-linux-amd64 /usr/local/bin/calicoctl
```

配置 `calicoctl` 以访问 Kubernetes。

```bash
echo "export KUBECONFIG=/root/.kube/config" >> ~/.bashrc
echo "export DATASTORE_TYPE=kubernetes" >> ~/.bashrc

echo $KUBECONFIG
echo $DATASTORE_TYPE
```

执行下面的命令，验证`calicoctl`能够访问数据库。

```bash
calicoctl get nodes -o wide
```

运行结果类似如下：

```console
NAME     ASN   IPV4   IPV6   
cka001                       
cka002                       
cka003  
```

节点是由 Kubernetes 节点对象支持的，因此我们应该看到与 `kubectl get nodes` 匹配的名称。

```bash
kubectl get nodes -o wide
```

运行结果：

```console
NAME     STATUS     ROLES                  AGE   VERSION   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cka001   NotReady   control-plane,master   23m   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
cka002   NotReady   <none>                 22m   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
cka003   NotReady   <none>                 21m   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
```

## 配置IP池

一个工作负载（workload）是容器或虚拟机，基于Calico的虚拟网络。
在Kubernetes中，工作负载是Pod。一个工作负载端点（endpoint）是工作负载用来连接Calico网络的虚拟网络接口。

IP池是Calico为工作负载端点使用的IP地址范围。

获取集群中当前的IP池。目前，在刚刚安装完之后，它是空的。

```bash
calicoctl get ippools
```

运行结果：

```console
NAME   CIDR   SELECTOR 
```

我们通过 `kubeadm init` 命令指定了 Pod CIDR 为 `10.244.0.0/16`。

现在，我们为集群创建两个 IP 池（IP pool），每个池之间不能重叠。

创建IP池`ipv4-ippool-1`: `10.244.0.0/18`

```bash
calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool-1
spec:
  cidr: 10.244.0.0/18
  ipipMode: Never
  natOutgoing: true
  disabled: false
  nodeSelector: all()
EOF
```

创建IP池`ipv4-ippool-2`: `10.244.192.0/19`

```bash
calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool-2
spec:
  cidr: 10.244.192.0/19
  ipipMode: Never
  natOutgoing: true
  disabled: true
  nodeSelector: all()
EOF
```

查询所创建的IP池。

```bash
calicoctl get ippools -o wide
```

运行结果：

```console
NAME            CIDR              NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
ipv4-ippool-1   10.244.0.0/18     true   Never      Never       false      false              all()      
ipv4-ippool-2   10.244.192.0/19   true   Never      Never       true       false              all()     
```

## 安装CNI插件

* 为插件创建Kubernetes用户账户。

Kubernetes使用容器网络接口（CNI）与像Calico这样的网络提供者进行交互。

以API形式呈现的，供Kubernetes使用的Calico二进制文件，称为CNI插件，必须安装在Kubernetes集群中的每个节点上。

CNI插件在创建Pod时与Kubernetes API服务器交互，既要获取附加信息，又要使用有关Pod的信息更新数据存储。

在Kubernetes *master*节点上，为CNI插件创建一个密钥以进行身份验证并签名证书请求。

切换到目录 `/etc/kubernetes/pki/`。

```bash
cd /etc/kubernetes/pki/
```

生成证书。

```bash
openssl req -newkey rsa:4096 \
  -keyout cni.key \
  -nodes \
  -out cni.csr \
  -subj "/CN=calico-cni"
```

使用主Kubernetes CA对此证书进行签名。

```bash
sudo openssl x509 -req -in cni.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out cni.crt \
  -days 3650
```

输出结果类似如下，用户是 `calico-cni`。

```console
Signature ok
subject=CN = calico-cni
Getting CA Private Key
```

赋予当前操作系统用户对文件`cni.crt`的操作权限。

```bash
sudo chown $(id -u):$(id -g) cni.crt
```

保持在`/etc/kubernetes/pki/`目录中，接下来我们将为CNI插件创建一个kubeconfig文件，用于访问Kubernetes。
将此`cni.kubeconfig`文件复制到集群中的每个节点。

```bash
APISERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')

echo $APISERVER

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=$APISERVER \
  --kubeconfig=cni.kubeconfig

kubectl config set-credentials calico-cni \
  --client-certificate=cni.crt \
  --client-key=cni.key \
  --embed-certs=true \
  --kubeconfig=cni.kubeconfig

kubectl config set-context cni@kubernetes \
  --cluster=kubernetes \
  --user=calico-cni \
  --kubeconfig=cni.kubeconfig

kubectl config use-context cni@kubernetes --kubeconfig=cni.kubeconfig
```

查询CNI上下文看起来类似下面的输出。

```bash
kubectl config get-contexts --kubeconfig=cni.kubeconfig
```

输出结果：

```console
CURRENT   NAME             CLUSTER      AUTHINFO     NAMESPACE
*         cni@kubernetes   kubernetes   calico-cni 
```

* 配置RBAC（ Role-Based Access Control）

为 CNI 插件的 Kubernetes 用户帐户配置 RBAC 角色和角色绑定。这将授予此用户帐户所需的 Kubernetes API 访问权限。

切换到home路径，作为当前工作目录。

```bash
cd ~
```

定义一个集群角色，CNI插件将使用该角色访问Kubernetes。

```bash
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-cni
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
 # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
      - clusterinformations
      - ippools
    verbs:
      - get
      - list
EOF
```

把上面创建的集群角色绑定到 `calico-cni` 用户账户。

```bash
kubectl create clusterrolebinding calico-cni --clusterrole=calico-cni --user=calico-cni
```

* 安装插件

提示：需要在每个节点上执行下面的安装步骤。

在`cka001`上安装。

以 **root** 用户运行下面的命令。

```bash
sudo su
```

安装CNI插件的二进制文件。
下载包链接：`https://github.com/projectcalico/cni-plugin/releases`和 `https://github.com/containernetworking/plugins/releases`。

```bash
mkdir -p /opt/cni/bin

curl -L -o /opt/cni/bin/calico https://github.com/projectcalico/cni-plugin/releases/download/v3.20.5/calico-amd64
chmod 755 /opt/cni/bin/calico

curl -L -o /opt/cni/bin/calico-ipam https://github.com/projectcalico/cni-plugin/releases/download/v3.20.5/calico-ipam-amd64
chmod 755 /opt/cni/bin/calico-ipam

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
tar xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin
```

创建配置文件目录。

```bash
mkdir -p /etc/cni/net.d/
```

复制前面生成的kubeconfig到我们创建的配置文件目录`/etc/cni/net.d/`下，改名为`calico-kubeconfig`，并修改其权限。

```bash
cp /etc/kubernetes/pki/cni.kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```

将下面的内容写入CNI配置文件`/etc/cni/net.d/10-calico.conflist`中。

```bash
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

将/etc/cni/net.d/calico-kubeconfig配置文件复制到当前操作系统用户（`root`）的home路径下。

```bash
cp /etc/cni/net.d/calico-kubeconfig ~
```

退出 `su` 的 root用户，返回常规用户，这里是`vagrant`。

```bash
exit
```

在节点`cka002`上安装。

当前仍然在节点`cka001`，通过sftp命令把生成的证书从节点`cka001`上传至节点`cka002`。

```bash
sftp -i cka-key-pair.pem cka002

put calico-amd64
put calicoctl-linux-amd64
put calico-ipam-amd64
put calico-kubeconfig
put cni-plugins-linux-amd64-v1.1.1.tgz
```

通过证书从节点`cka001`登录到节点`cka002`。

```bash
ssh -i cka-key-pair.pem cka002
```

创建目录`/opt/cni/bin`以存放cni二进制文件。

```bash
mkdir -p /opt/cni/bin

cp calico-amd64 /opt/cni/bin/calico
cp calico-ipam-amd64 /opt/cni/bin/calico-ipam

tar xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin

mkdir -p /etc/cni/net.d/

cp calico-kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```

更新CNI配置文件`/etc/cni/net.d/10-calico.conflist`。

```bash
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

返回至节点 `cka001`。

```bash
exit
```

在`cka003`上安装。

类似`cka002`的安装过程，从节点`cka001`上传证书到节点`cka003`，并登录到节点`cka003`完成下面的命令。

```bash
sftp -i cka-key-pair.pem cka003
```

```bash
put calico-amd64
put calicoctl-linux-amd64
put calico-ipam-amd64
put calico-kubeconfig
put cni-plugins-linux-amd64-v1.1.1.tgz
```

```bash
ssh -i cka-key-pair.pem cka003
```

```bash
mkdir -p /opt/cni/bin

cp calico-amd64 /opt/cni/bin/calico
cp calico-ipam-amd64 /opt/cni/bin/calico-ipam

tar xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin

mkdir -p /etc/cni/net.d/

cp calico-kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```

```bash
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

返回至节点`cka001`。

```bash
exit
```

当前工作目录仍然是节点`cka001`的home目录。

至此，此时，Kubernetes 节点将变为 Ready 状态，因为 Kubernetes 已安装了网络提供程序和配置。

```bash
kubectl get nodes
```

运行结果：

```console
NAME     STATUS   ROLES                  AGE     VERSION
cka001   Ready    control-plane,master   4h50m   v1.24.0
cka002   Ready    <none>                 4h49m   v1.24.0
cka003   Ready    <none>                 4h49m   v1.24.0
```

## 安装Typha

Typha 处于 Kubernetes API 服务器和每个节点守护进程（如运行在 calico/node 中的 Felix 和 confd）之间。
它监视这些守护进程使用的 Kubernetes 资源和 Calico 自定义资源，每当资源更改时，它会将更新扩散到这些守护进程。
这减少了 Kubernetes API 服务器需要服务的监视数，提高了集群的可扩展性。

* 准备证书

下面，我们使用相互认证的TLS来确保calico/node和Typha之间的通信安全。
生成一个证书授权机构（CA）并使用它来为Typha签署证书。

将当前工作目录改为 `/etc/kubernetes/pki/`。

```bash
cd /etc/kubernetes/pki/
```

创建CA证书和密钥。

```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout typhaca.key \
  -nodes \
  -out typhaca.crt \
  -subj "/CN=Calico Typha CA" \
  -days 365
```

把CA证书存放在ConfigMap中，使Typha和calico/node能够访问。

```bash
kubectl create configmap -n kube-system calico-typha-ca --from-file=typhaca.crt
```

生成Typha密钥和证书签名请求（certificate signing request，CSR）。

```bash
openssl req -newkey rsa:4096 \
  -keyout typha.key \
  -nodes \
  -out typha.csr \
  -subj "/CN=calico-typha"
```

证书的通用名称（CN）设置为 `calico-typha`。`calico/node` 将被用来验证此名称。

使用 CA 对 Typha 证书进行签名。

```bash
openssl x509 -req -in typha.csr \
  -CA typhaca.crt \
  -CAkey typhaca.key \
  -CAcreateserial \
  -out typha.crt \
  -days 365
```

运行结果：

```console
Signature ok
subject=CN = calico-typha
Getting CA Private Key
```

将 Typha 密钥和证书存储在一个 secret 中，以便 Typha 可以访问。

```bash
kubectl create secret generic -n kube-system calico-typha-certs --from-file=typha.key --from-file=typha.crt
```

* 配置RBAC

当前工作目录为home路径。

```bash
cd ~
```

创建一个Typha使用的ServiceAccount。

```bash
kubectl create serviceaccount -n kube-system calico-typha
```

为 Typha 创建一个集群角色，有观察 Calico 数据存储对象的权限。

```bash
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-typha
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
      - endpoints
      - services
      - nodes
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - globalfelixconfigs
      - felixconfigurations
      - bgppeers
      - globalbgpconfigs
      - bgpconfigurations
      - ippools
      - ipamblocks
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - clusterinformations
      - hostendpoints
      - blockaffinities
      - networksets
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      #- ippools
      #- felixconfigurations
      - clusterinformations
    verbs:
      - get
      - create
      - update
EOF
```

将创建的集群角色绑定到`calico-typha`这个ServiceAccount。

```bash
kubectl create clusterrolebinding calico-typha --clusterrole=calico-typha --serviceaccount=kube-system:calico-typha
```

* 安装Deployment

由于 `calico/node` 需要 Typha，而 `calico/node` 负责建立 Pod 网络，所以我们把 Typha 作为主机网络的 Pod 运行，以避免鸡生蛋或蛋生鸡的问题（chicken-and-egg problem）。

我们运行 3 个 Typha 副本，这样即使在滚动更新期间发生单个故障，也不会使 Typha 不可用。

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      k8s-app: calico-typha
  template:
    metadata:
      labels:
        k8s-app: calico-typha
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: 'true'
    spec:
      hostNetwork: true
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
      serviceAccountName: calico-typha
      priorityClassName: system-cluster-critical
      containers:
      - image: calico/typha:v3.8.0
        name: calico-typha
        ports:
        - containerPort: 5473
          name: calico-typha
          protocol: TCP
        env:
          # Disable logging to file and syslog since those don't make sense in Kubernetes.
          - name: TYPHA_LOGFILEPATH
            value: "none"
          - name: TYPHA_LOGSEVERITYSYS
            value: "none"
          # Monitor the Kubernetes API to find the number of running instances and rebalance
          # connections.
          - name: TYPHA_CONNECTIONREBALANCINGMODE
            value: "kubernetes"
          - name: TYPHA_DATASTORETYPE
            value: "kubernetes"
          - name: TYPHA_HEALTHENABLED
            value: "true"
          # Location of the CA bundle Typha uses to authenticate calico/node; volume mount
          - name: TYPHA_CAFILE
            value: /calico-typha-ca/typhaca.crt
          # Common name on the calico/node certificate
          - name: TYPHA_CLIENTCN
            value: calico-node
          # Location of the server certificate for Typha; volume mount
          - name: TYPHA_SERVERCERTFILE
            value: /calico-typha-certs/typha.crt
          # Location of the server certificate key for Typha; volume mount
          - name: TYPHA_SERVERKEYFILE
            value: /calico-typha-certs/typha.key
        livenessProbe:
          httpGet:
            path: /liveness
            port: 9098
            host: localhost
          periodSeconds: 30
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /readiness
            port: 9098
            host: localhost
          periodSeconds: 10
        volumeMounts:
        - name: calico-typha-ca
          mountPath: "/calico-typha-ca"
          readOnly: true
        - name: calico-typha-certs
          mountPath: "/calico-typha-certs"
          readOnly: true
      volumes:
      - name: calico-typha-ca
        configMap:
          name: calico-typha-ca
      - name: calico-typha-certs
        secret:
          secretName: calico-typha-certs
EOF
```

我们设置 `TYPHA_CLIENTCN` 为 `calico-node`，后续将用于 `calico/node` 证书的通用名称。

确认 Typha 已经启动并运行了三个实例。

```bash
kubectl get pods -l k8s-app=calico-typha -n kube-system
```

运行结果：

```console
NAME                           READY   STATUS    RESTARTS   AGE
calico-typha-5b8669646-b2xnq   1/1     Running   0          20s
calico-typha-5b8669646-q5glk   0/1     Pending   0          20s
calico-typha-5b8669646-rvv86   1/1     Running   0          20s
```

遇到如下错误信息。

```console
0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't have free ports for the requested pod ports.
```

* 安装Service

`calico/node`使用Kubernetes Service以获得对Typha的负载均衡访问。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  ports:
    - port: 5473
      protocol: TCP
      targetPort: calico-typha
      name: calico-typha
  selector:
    k8s-app: calico-typha
EOF
```

验证Typha正在使用TLS。

```bash
TYPHA_CLUSTERIP=$(kubectl get svc -n kube-system calico-typha -o jsonpath='{.spec.clusterIP}')
curl https://$TYPHA_CLUSTERIP:5473 -v --cacert /etc/kubernetes/pki/typhaca.crt
```

运行结果：

```console
*   Trying 11.244.91.165:5473...
* TCP_NODELAY set
* Connected to 11.244.91.165 (11.244.91.165) port 5473 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/kubernetes/pki/typhaca.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS alert, bad certificate (554):
* error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
* Closing connection 0
curl: (35) error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
```

上面的错误信息说明 Typha 正在使用 TLS 证书，且因我们没有提供证书而拒绝连接。
下面我们会使用一个证书部署 calico/node，这样 Typha 就会接受连接了。

## 安装calico/node

`calico/node` 运行三个守护进程。

* Felix，Calico的每个节点守护进程
* BIRD，一个守护进程，使用BGP协议与其他节点交换路由信息
* confd，一个守护进程，监视Calico数据存储中的配置更改，并更新BIRD的配置文件

* 准备证书

切换到目录 `/etc/kubernetes/pki/`。

```bash
cd /etc/kubernetes/pki/
```

创建 `calico/node` 密钥，用于认证 Typha 和证书签名请求 (certificate signing request，CSR)。

```bash
openssl req -newkey rsa:4096 \
  -keyout calico-node.key \
  -nodes \
  -out calico-node.csr \
  -subj "/CN=calico-node"
```

这个证书的公共名称 (CN) 是 `calico-node`，这是我们在上一个演示中配置 Typha 接受的名称。

使用我们前面创建的 CA 对 Felix 证书进行签名。

```bash
openssl x509 -req -in calico-node.csr \
  -CA typhaca.crt \
  -CAkey typhaca.key \
  -CAcreateserial \
  -out calico-node.crt \
  -days 365
```

运行结果：

```console
Signature ok
subject=CN = calico-node
Getting CA Private Key
```

将密钥和证书存储在 calico/node 将要访问的 Secret 中。

```bash
kubectl create secret generic -n kube-system calico-node-certs --from-file=calico-node.key --from-file=calico-node.crt
```

* 准备RBAC

切换至当前用户的home目录。

```bash
cd ~
```

创建一个 `calico/node` 要使用的 ServiceAccount。

```bash
kubectl create serviceaccount -n kube-system calico-node
```

准备一个集群角色，该角色具有读写Calico数据库对象的权限。

```bash
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-node
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # EndpointSlices are used for Service-based network policy rule
  # enforcement.
  - apiGroups: ["discovery.k8s.io"]
    resources:
      - endpointslices
    verbs:
      - watch
      - list
  - apiGroups: [""]
    resources:
      - endpoints
      - services
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
      # Used to discover Typhas.
      - get
  # Pod CIDR auto-detection on kubeadm needs access to config maps.
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      # Needed for clearing NodeNetworkUnavailable flag.
      - patch
      # Calico stores some configuration information in node annotations.
      - update
  # Watch for changes to Kubernetes NetworkPolicies.
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  # Used by Calico for policy information.
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
    verbs:
      - list
      - watch
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
  # Used for creating service account tokens to be used by the CNI plugin
  - apiGroups: [""]
    resources:
      - serviceaccounts/token
    resourceNames:
      - calico-node
    verbs:
      - create
  # Calico monitors various CRDs for config.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - globalfelixconfigs
      - felixconfigurations
      - bgppeers
      - globalbgpconfigs
      - bgpconfigurations
      - ippools
      - ipamblocks
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - networksets
      - clusterinformations
      - hostendpoints
      - blockaffinities
    verbs:
      - get
      - list
      - watch
  # Calico must create and update some CRDs on startup.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ippools
      - felixconfigurations
      - clusterinformations
    verbs:
      - create
      - update
  # Calico stores some configuration information on the node.
  - apiGroups: [""]
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
  # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
    verbs:
      - get
  # Block affinities must also be watchable by confd for route aggregation.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
    verbs:
      - watch
EOF
```

把创建的集群角色绑定到`calico-node`这个ServiceAccount。

```bash
kubectl create clusterrolebinding calico-node --clusterrole=calico-node --serviceaccount=kube-system:calico-node
```

* 安装daemonset

切换至当前用户的home目录。

```bash
cd ~
```

`calico/node`作为daemonset运行，安装在群集中的每个节点上。

修改`image: calico/node:v3.20.0`为实际安装版本。

创建daemonset。

```bash
kubectl apply -f - <<EOF
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      hostNetwork: true
      tolerations:
        # Make sure calico-node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-node
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      containers:
        # Runs calico-node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: calico/node:v3.20.0
          env:
            # Use Kubernetes API as the backing datastore.
            - name: DATASTORE_TYPE
              value: "kubernetes"
            - name: FELIX_TYPHAK8SSERVICENAME
              value: calico-typha
            # Wait for the datastore.
            - name: WAIT_FOR_DATASTORE
              value: "true"
            # Set based on the k8s node name.
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              value: bird
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            # Disable file logging so kubectl logs works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            - name: FELIX_HEALTHENABLED
              value: "true"
            # Location of the CA bundle Felix uses to authenticate Typha; volume mount
            - name: FELIX_TYPHACAFILE
              value: /calico-typha-ca/typhaca.crt
            # Common name on the Typha certificate; used to verify we are talking to an authentic typha
            - name: FELIX_TYPHACN
              value: calico-typha
            # Location of the client certificate for connecting to Typha; volume mount
            - name: FELIX_TYPHACERTFILE
              value: /calico-node-certs/calico-node.crt
            # Location of the client certificate key for connecting to Typha; volume mount
            - name: FELIX_TYPHAKEYFILE
              value: /calico-node-certs/calico-node.key
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          lifecycle:
            preStop:
              exec:
                command:
                - /bin/calico-node
                - -shutdown
          livenessProbe:
            httpGet:
              path: /liveness
              port: 9099
              host: localhost
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -bird-ready
              - -felix-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /run/xtables.lock
              name: xtables-lock
              readOnly: false
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /var/run/nodeagent
              name: policysync
            - mountPath: "/calico-typha-ca"
              name: calico-typha-ca
              readOnly: true
            - mountPath: /calico-node-certs
              name: calico-node-certs
              readOnly: true
      volumes:
        # Used by calico-node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
        # Used to create per-pod Unix Domain Sockets
        - name: policysync
          hostPath:
            type: DirectoryOrCreate
            path: /var/run/nodeagent
        - name: calico-typha-ca
          configMap:
            name: calico-typha-ca
        - name: calico-node-certs
          secret:
            secretName: calico-node-certs
EOF
```

验证在集群中每个节点上 `calico/node` 是否在运行，安装后，一般在几分钟内会变为 Running 状态。

```bash
kubectl get pod -l k8s-app=calico-node -n kube-system
```

运行结果：

```console
NAME                READY   STATUS    RESTARTS   AGE
calico-node-4c4sp   1/1     Running   0          40s
calico-node-j2z6v   1/1     Running   0          40s
calico-node-vgm9n   1/1     Running   0          40s
```

## 测试网络

### pod之间的ping

创建三个busybox实例。

```bash
kubectl create deployment pingtest --image=busybox --replicas=3 -- sleep infinity
```

查询他们的IP地址。

```bash
kubectl get pods --selector=app=pingtest --output=wide
```

运行结果：

```console
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
pingtest-585b76c894-chwjq   1/1     Running   0          7s    10.244.31.1    cka002   <none>           <none>
pingtest-585b76c894-s2tbs   1/1     Running   0          7s    10.244.31.0    cka002   <none>           <none>
pingtest-585b76c894-vm9wn   1/1     Running   0          7s    10.244.28.64   cka003   <none>           <none>
```

留意第二个和第三个 pod 的 IP 地址。
随后我们会在第一个 pod 中运行 exec 命令。
在第一个 pod 内部，对另外两个 pod 的 IP 地址进行 ping 测试。

例如：

```bash
kubectl exec -ti pingtest-585b76c894-chwjq -- sh
/ # ping 10.244.31.1 -c 4
4 packets transmitted, 4 packets received, 0% packet loss

/ # ping 10.244.31.0 -c 4
4 packets transmitted, 4 packets received, 0% packet loss

/ # ping 10.244.28.64 -c 4
4 packets transmitted, 0 packets received, 100% packet loss
```

### 路由检查

从其中一个节点验证是否能ping通到每个pod的IP地址。例如：

```bash
ip route get 10.244.31.1
ip route get 10.244.31.0
ip route get 10.244.28.64
```

在上面的结果中，示例中的 `via <cka001_ip>` （它是控制平面）表示此Pod IP的下一跳，这与Pod所在节点的IP地址匹配，符合预期。

不同IP池的IPAM分配。在前面的演示中，我们创建了两个IP池，但将一个禁用了。

```bash
calicoctl get ippools -o wide
```

运行结果：

```console
NAME            CIDR              NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
ipv4-ippool-1   10.244.0.0/18     true   Never      Never       false      false              all()      
ipv4-ippool-2   10.244.192.0/19   true   Never      Never       true       false              all()   
```

激活第二个IP池。

```bash
calicoctl --allow-version-mismatch apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool-2
spec:
  cidr: 10.244.192.0/19
  ipipMode: Never
  natOutgoing: true
  disabled: false
  nodeSelector: all()
EOF
```

查询IP池的状态。

```bash
calicoctl get ippools -o wide
```

运行结果：

```console
NAME            CIDR              NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
ipv4-ippool-1   10.244.0.0/18     true   Never      Never       false      false              all()      
ipv4-ippool-2   10.244.192.0/19   true   Never      Never       false      false              all()      
```

创建一个 Pod，以显式方式请求从 `pool2` 分配一个 IP 地址。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pingtest-ippool-2
  annotations:
    cni.projectcalico.org/ipv4pools: "[\"ipv4-ippool-2\"]"
spec:
  containers:
  - args:
    - sleep
    - infinity
    image: busybox
    imagePullPolicy: Always
    name: pingtest
EOF
```

验证pod已经从 `pool2` 分配一个 IP 地址。

```bash
kubectl get pod pingtest-ippool-2 -o wide
```

运行结果：

```console
NAME                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
pingtest-ippool-2   1/1     Running   0          18s   10.244.203.192   cka003   <none>           <none>
```

连接并进入Pod `pingtest-585b76c894-chwjq`内部。

```bash
kubectl exec -ti pingtest-585b76c894-chwjq -- sh
/ # 10.244.203.192 -c 4
4 packets transmitted, 0 packets received, 100% packet loss
```

标记：
演示止于此，路由没有安装预期工作，原因查找中。

删除演示中创建的临时资源。

```bash
kubectl delete deployments.apps pingtest
kubectl delete pod pingtest-ippool-2
```

参考：[End-to-end Calico installation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/)
