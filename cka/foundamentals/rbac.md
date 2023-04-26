# CKA自学笔记18:RBAC鉴权

## 摘要

演示场景：

1. 为一个集群创建不同的配置文件。
2. 使用 `cfssl` 为每个配置文件生成证书。
3. 创建新的 `kubeconfig` 文件，包含所有配置文件和相应的用户。
4. 将旧的和新的 `kubeconfig` 文件合并到新的 `kubeconfig` 文件中。我们可以切换不同的上下文来进行进一步的演示。

基本概念

* 基于角色的访问控制（Role-based access control，RBAC）是一种基于组织中个人用户角色的访问计算机或网络资源的方法。
* 当使用客户端证书认证时，我们可以通过 easyrsa、openssl 或 cfssl 手动生成证书。

建议：

* kubeconfig 的目的是为不同用户授予不同集群的权限。
* 不同的上下文将链接到不同的集群。
* 不建议将多个用户的上下文放在一个 kubeconfig 中使用同一个集群。
* 建议为一个用户使用一个 kubeconfig 文件。

## 安装cfssl

安装 `cfssl`。

```bash
apt install golang-cfssl
```

## 设置多个上下文

### 当前上下文

执行命令 `kubectl config` 查询当前使用的上下文（contenxt）。

```bash
kubectl config get-contexts
```

我们可以得到类似如下集群的关键信息。

* 集群名称：kubernetes
* 系统账号：kubenetes-admin
* 当前上下文名称：kubernetes-admin@kubernetes（格式为 `<system_account>@<cluster_name>`）

```console
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```

### 创建CA配置文件

CA（Certificate Authority）配置文件（config file）是一个用于存储证书颁发机构（CA）信息的文件。
在使用TLS/SSL加密通信时，需要使用证书来验证通信双方的身份。
而CA则是负责签发和验证证书的机构，因此在建立TLS/SSL连接时需要先验证CA的信任关系，以保证证书的有效性。
CA文件中存储了CA的公钥信息和其他相关的配置信息。
在Kubernetes中，使用CA文件来验证证书的有效性和授权访问。

查看目录`/etc/kubernetes/pki`及其子目录的结构情况。

```bash
tree /etc/kubernetes/pki
```

运行结果，下面是Kubernetes初始安装后的文件结构的示例内容。

```console
/etc/kubernetes/pki
├── apiserver.crt
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── etcd
│   ├── ca.crt
│   ├── ca.key
│   ├── healthcheck-client.crt
│   ├── healthcheck-client.key
│   ├── peer.crt
│   ├── peer.key
│   ├── server.crt
│   └── server.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
└── sa.pub
```

进入目录`/etc/kubernetes/pki`，即当前工作目录。

```bash
cd /etc/kubernetes/pki
```

检查文件`ca-config.json`是否已经存在与当前工作目录。

```bash
ll ca-config.json
```

如果不存在，则创建这个文件。

* 我们可以添加多个配置文件来指定不同的过期日期、场景、参数等。
* 配置文件将用于签署证书。
* `87600`小时大约是10年。

这里我们将在`ca-config.json`文件中添加一个名为`dev`的配置文件。

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "dev": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

### 创建证书签名请求 (CSR) 文件

证书签名请求（CSR）资源用于请求由指定签名者签署的证书，此后可以在最终签署之前批准或拒绝请求。

设置CSR的`CN`和`O`属性很重要。

* `CN`是请求CSR的用户的名称。
* `O`是此用户将属于的组。我们可以参考RBAC以了解标准组。

保持当前工作目录为`/etc/kubernetes/pki`。

创建CSR文件`cka-dev-csr.json`。
`CN` 代表 `cka-dev`。
`O` 代表 `k8s`。

```bash
cat > cka-dev-csr.json <<EOF
{
  "CN": "cka-dev",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

为我们之前定义的配置文件生成证书和密钥。
使用`cfssljson -bare cka-dev`命令会生成两个文件，`cka-dev.pem`作为公钥，`cka-dev-key.pem`作为私钥。

```bash
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=ca-config.json -profile=dev cka-dev-csr.json | cfssljson -bare cka-dev
```

验证这2个文件已经成功创建出来了。

```bash
ll -tr | grep cka-dev
```

运行结果：

```console
-rw-r--r-- 1 root root  222 Jul 24 08:49 cka-dev-csr.json
-rw-r--r-- 1 root root 1281 Jul 24 09:14 cka-dev.pem
-rw------- 1 root root 1675 Jul 24 09:14 cka-dev-key.pem
-rw-r--r-- 1 root root 1001 Jul 24 09:14 cka-dev.csr
```

### 创建kubeconfig文件

获取控制平面的IP地址（如：`<cka001_ip>`）来拼接出环境变量`APISERVER`的值，如：`https://<control_plane_ip>:<port>`。

```bash
kubectl get node -owide
```

运行结果：

```bash
NAME     STATUS   ROLES                  AGE   VERSION  OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cka001   Ready    control-plane,master   14h   v1.24.0  Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
cka002   Ready    <none>                 14h   v1.24.0  Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
cka003   Ready    <none>                 14h   v1.24.0  Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
```

设定并输出环境变量`APISERVER`。

```bash
echo "export APISERVER=\"https://<cka001_ip>:6443\"" >> ~/.bashrc
source ~/.bashrc
```

验证环境变量`APISERVER`的值。

```bash
echo $APISERVER
```

运行结果：

```bash
https://<cka001_ip>:6443
```

1. 设置集群

保持当前工作目录为 `/etc/kubernetes/pki`。

生成`kubeconfig`文件。

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${APISERVER} \
  --kubeconfig=cka-dev.kubeconfig
```

现在我们生成了新的配置文件 `cka-dev.kubeconfig`。

```bash
ll -tr | grep cka-dev
```

输出结果：

```bash
-rw-r--r-- 1 root root  222 Jul 24 08:49 cka-dev-csr.json
-rw-r--r-- 1 root root 1281 Jul 24 09:14 cka-dev.pem
-rw------- 1 root root 1675 Jul 24 09:14 cka-dev-key.pem
-rw-r--r-- 1 root root 1001 Jul 24 09:14 cka-dev.csr
-rw------- 1 root root 1671 Jul 24 09:16 cka-dev.kubeconfig
```

读取配置文件`cka-dev.kubeconfig`的内容。

```bash
cat cka-dev.kubeconfig
```

输出的内容：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <your_key>
    server: https://<cka001_ip>:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

2. 配置用户

在文件`cka-dev.kubeconfig`中，用户信息这部分是空的。

下面我们配置一个用户`cka-dev`。

```bash
kubectl config set-credentials cka-dev \
  --client-certificate=/etc/kubernetes/pki/cka-dev.pem \
  --client-key=/etc/kubernetes/pki/cka-dev-key.pem \
  --embed-certs=true \
  --kubeconfig=cka-dev.kubeconfig
```

现在，用户信息已经被添加到配置文件`cka-dev.kubeconfig`中了。

```bash
cat cka-dev.kubeconfig
```

输出结果：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <your_key>
    server: https://<cka001_ip>:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: cka-dev
  user:
    client-certificate-data: <your_key>
    client-key-data: <your_key>
```

至此我们得到了一个完整的配置文件`cka-dev.kubeconfig`。
由于我们没有在 kubeconfig 文件中设置当前上下文，当我们使用它来获取节点信息时，会收到以下错误。

```bash
kubectl --kubeconfig=cka-dev.kubeconfig get nodes
```

运行结果：

```console
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

当前上下文内容是空白。

```bash
kubectl --kubeconfig=cka-dev.kubeconfig config get-contexts
```

输出结果：

```console
CURRENT   NAME   CLUSTER   AUTHINFO   NAMESPACE
```

3. 配置上下文

配置上下文。

```bash
kubectl config set-context dev --cluster=kubernetes --user=cka-dev  --kubeconfig=cka-dev.kubeconfig
```

现在我们配置了上下文，但其中`CURRENT`仍然是空白。

```bash
kubectl --kubeconfig=cka-dev.kubeconfig config get-contexts
```

运行结果：

```console
CURRENT   NAME   CLUSTER      AUTHINFO   NAMESPACE
          dev    kubernetes   cka-dev 
```

设置默认上下文。上下文将为多集群环境中的集群和用户链接，并且我们可以切换到不同的集群。

```bash
kubectl --kubeconfig=cka-dev.kubeconfig config use-context dev
```

4. 验证

现在`CURRENT`已经被标记为`*`了，这就说明当前上下文已经配置好了。

```bash
kubectl --kubeconfig=cka-dev.kubeconfig config get-contexts
```

运行结果：

```console
CURRENT   NAME   CLUSTER      AUTHINFO   NAMESPACE
*         dev    kubernetes   cka-dev      
```

因为用户 `cka-dev` 在该集群中没有授权，所以当我们尝试获取 Pod 或 Node 的信息时，会收到“禁止访问（forbidden）”错误。

```bash
kubectl --kubeconfig=/etc/kubernetes/pki/cka-dev.kubeconfig get pod 
kubectl --kubeconfig=/etc/kubernetes/pki/cka-dev.kubeconfig get node
```

### Merge kubeconfig files

Make a copy of your existing config

```console
cp ~/.kube/config ~/.kube/config.old 
```

Merge the two config files together into a new config file `/tmp/config`.

```console
KUBECONFIG=~/.kube/config:/etc/kubernetes/pki/cka-dev.kubeconfig  kubectl config view --flatten > /tmp/config
```

Replace the old config with the new merged config

```console
mv /tmp/config ~/.kube/config
```

Now the new `~/.kube/config` looks like below.

```console
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <your_key>
    server: https://<cka001_ip>:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: cka-dev
  name: dev
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: cka-dev
  user:
    client-certificate-data: <your_key>
    client-key-data: <your_key>
- name: kubernetes-admin
  user:
    client-certificate-data: <your_key>
    client-key-data: <your_key>
```

Verify contexts after kubeconfig merged.

```console
kubectl config get-contexts
```

Current context is the system default `kubernetes-admin@kubernetes`.

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          dev                           kubernetes   cka-dev            
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```

### Namespaces & Contexts

Get list of Namespace with Label information.

```console
kubectl get ns --show-labels
```

Create Namespace `cka`.

```console
kubectl create namespace cka
```

Use below command to set a context with new update, e.g, update default namespace, etc..

```console
kubectl config set-context <context name> --cluster=<cluster name> --namespace=<namespace name> --user=<user name> 
```

Let's set default namespace to each context.

```console
kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --namespace=default --user=kubernetes-admin
kubectl config set-context dev --cluster=kubernetes --namespace=cka --user=cka-dev
```

Let's check current context information.

```console
kubectl config get-contexts
```

Output:

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          dev                           kubernetes   cka-dev            cka
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```

To switch to a new context, use below command.

```console
kubectl config use-contexts <context name>
```

For example.

```console
kubectl config use-context dev
```

Verify if it's changed as expected.

```console
kubectl config get-contexts
```

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         dev            kubernetes   cka-dev            cka
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```

Be noted, four users beginning with `cka-dev` created don't have any authorizations, e.g., access namespaces, get pods, etc..
Referring RBAC to grant their authorizations.

### Role & RoleBinding

Switch to context `kubernetes-admin@kubernetes`.

```console
kubectl config use-context kubernetes-admin@kubernetes
```

Use `kubectl create role` command  with option `--dry-run=client` and `-o yaml` to generate yaml template for customizing.

```console
kubectl create role admin-dev --resource=pods --verb=get --verb=list --verb=watch --dry-run=client -o yaml
```

Create role `admin-dev` on namespace `cka`.

```console
kubectl apply -f - << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: cka
  name: admin-dev
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
EOF
```

Use `kubectl create rolebinding` command  with option `--dry-run=client` and `-o yaml` to generate yaml template for customizing.

```console
kubectl create rolebinding admin --role=admin-dev --user=cka-dev --dry-run=client -o yaml
```

Create rolebinding `admin` on namespace `cka`.

```console
kubectl apply -f - << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin
  namespace: cka
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: admin-dev
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: cka-dev
EOF
```

Verify authorzation of user `cka-dev` on Namespace `cka`.

Switch to context `dev`.

```console
kubectl config use-context dev
```

Get Pods status in Namespace `cka`. Success!

```console
kubectl get pod -n cka
```

Get Pods status in Namespace `kube-system`. Failed, because the authorzation is only for Namespace `cka`.

```console
kubectl get pod -n kube-system
```

Get Nodes status. Failed, because the role we defined is only for Pod resource.

```console
kubectl get node
```

Create a Pod in Namespace `dev`. Failed because we only have `get`,`watch`,`list` for Pod, no `create` authorization.

```console
kubectl run nginx --image=nginx -n cka
```

### ClusterRole & ClusterRoleBinding

Switch to context `kubernetes-admin@kubernetes`.

```console
kubectl config use-context kubernetes-admin@kubernetes
```

Create a ClusterRole `nodes-admin` with authorization `get`,`watch`,`list` for `nodes` resource.

```console
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nodes-admin
rules:
- apiGroups:
  - ""
  resources: 
  - nodes
  verbs:
  - get
  - watch
  - list
EOF
```

Bind ClusterRole `nodes-admin` to user `cka-dev`.

```console
kubectl apply -f - << EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
subjects:
- kind: User
  name: cka-dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: nodes-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

Verify Authorization

Switch to context `dev`.

```console
kubectl config use-context dev
```

Get node information. Success!

```console
kubectl get node
```

Switch to system context.

```console
kubectl config use-context kubernetes-admin@kubernetes 
```

### ClusterRole and ServiceAccount

!!! Scenario
    *Create a ClusterRole, which is authorized to create Deployment, StatefulSet, DaemonSet.
    * Bind the ClusterRole to a ServiceAccount.

Demo:

```console
kubectl create namespace my-namespace

kubectl -n my-namespace create serviceaccount my-sa

kubectl create clusterrole my-clusterrole --verb=create --resource=deployments,statefulsets,daemonsets

kubectl -n my-namespace create rolebinding my-clusterrolebinding --clusterrole=my-clusterrole --serviceaccount=my-namespace:my-sa
```

Clean up.

```console
kubectl delete namespace my-namespace 
kubectl delete clusterrole my-clusterrole
```

!!! Hints
    1. A RoleBinding may reference any Role in the same namespace.
    2. A RoleBinding can reference a ClusterRole and bind that ClusterRole to the namespace of the RoleBinding.
    3. If you want to bind a ClusterRole to all the namespaces in your cluster, you use a ClusterRoleBinding.
    4. Use RoleBinding to bind ClusterRole is to reuse the ClusterRole for namespaced resources, avoid duplicated namespaced roles for same authorization.
