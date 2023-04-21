# CKA自学笔记7:kubectl基础

## 摘要

了解如何使用`kubectl`操作Kubernetes集群。

- via [API](https://kubernetes.io/docs/reference/kubernetes-api/)
- via kubectl
- via Dashboard

## 检查当前kubeconfig文件配置

通过命令 `kubectl config` 检查当前配置文件中的上下文。

```bash
echo $KUBECONFIG
kubectl config view
kubectl config get-contexts
```

## 获取资源清单

读取所有支持的资源清单。

```bash
kubectl api-resources
```

## 获取集群状态

Kubernetes 控制面板运行在 `https://<control_plane_ip>:6443`。

CoreDNS 运行在 `https://<control_plane_ip>:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy`。

```bash
kubectl cluster-info
kubectl cluster-info dump
```

## 读取当前资源

执行命令 `kubectl get --help` 可以得到get命令的示例和使用方法。

读取当前控制面板的健康状态。

```bash
kubectl get componentstatuses
kubectl get cs
```

运行结果：

```console
NAME                 STATUS    MESSAGE                         ERROR
etcd-0               Healthy   {"health":"true","reason":""}   
scheduler            Healthy   ok                              
controller-manager   Healthy   ok 
```

## 读取节点状态和信息

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node cka001
```

可以通过命令 `kubectl create --help` 来获取get命令的帮助和示例。

## 创建Namespace

```bash
kubectl create namespace --help
kubectl create namespace my-namespace
```

提示：

命名空间Namespace是一个集群，包含了服务。服务可能在一个节点上，也可能不在。

- Namespace是一种用来组织服务的方式，它可以对服务进行隔离和划分。
- 不同的Namespace下，可以存在相同的服务名，但是不同的Namespace之间的服务不能直接通信，需要通过Service或Ingress来暴露。
- 服务是一种提供功能的实体
- 节点是一种运行服务的物理或虚拟机器

## 创建deployment

在某个Namespace中创建Deployment。

```bash
kubectl -n my-namespace create deployment my-busybox \
  --image=busybox \
  --replicas=3 \
  --port=5701
```

## 创建ClusterRole

```bash
kubectl create clusterrole --help

kubectl create clusterrole pod-creater \
  -n my-namespace \
  --verb=create \
  --resource=deployment \
  --resource-name=my-busybox
```

## 创建ServiceAccount

```bash
kubectl create serviceaccount --help
kubectl -n my-namespace create serviceaccount my-service-account
```

## 创建RoleBinding

 `RoleBinding`可以引用同一命名空间中的一个Role，或者全局命名空间中的一个ClusterRole。

- `RoleBinding`是一种用来授权角色的资源。
- Role是一种定义权限的资源，只能在同一命名空间内生效。
- ClusterRole是一种定义权限的资源，可以在整个集群内生效。

```bash
kubectl create rolebinding --help

kubectl create rolebinding NAME \
  --clusterrole=NAME|--role=NAME \
  [--user=username] \
  [--group=groupname] \
  [--serviceaccount=namespace:serviceaccountname] \
  [--dry-run=server|client|none]

kubectl create rolebinding my-admin \
  --clusterrole=pod-creater \
  --serviceaccount=my-namespace:my-service-account
```

## 使用proxy

我们可以使用`kubectl proxy`命令来打开一个到API服务器的隧道（tunnel），并使它在本地可用 - 通常是在`localhost:8001` / `127.0.0.1:8001`。当我想要使用API时，最简单的方法就是获取访问权限。

运行命令`kubectl proxy &`并在浏览器中打开`http://localhost:8001/api/v1`。 只打开`http://localhost:8001`会返回错误，因为我们只能访问API的某些内容，因此API路径很重要。

要点是：

- `kubectl proxy`命令可以创建一个本地代理，让我们可以访问API服务器。
- API服务器提供了集群的各种信息和操作。
- 我们需要指定正确的API路径，才能访问我们想要的资源。

```bash
kubectl proxy &
```

输出结果：

```console
[1] 102358
Starting to serve on 127.0.0.1:8001
```

比如：

```http
http://127.0.0.1:8001/
http://127.0.0.1:8001/api/v1
http://127.0.0.1:8001/api/v1/namespaces
http://127.0.0.1:8001/api/v1/namespaces/default
http://127.0.0.1:8001/api/v1/namespaces/sock-shop/pods
```

## 作为应用程序访问

如果我们作为应用程序而不是管理员来访问kubernetes，就不能使用`kubectl`，可以用`curl`程序来代替`kubectl`。 我们必须向集群发送HTTP请求，询问可用的节点。

确保`kubectl proxy`正在运行，并在`http://localhost:8001/`上提供服务。

执行下面的命令时加上一个`-v=9`的标志，它会显示所有需要的信息。

要点：

- 访问（access）是一种获取资源或服务的行为。

- 应用程序（application）是一种执行特定功能的软件。

- 作为应用程序访问意味着使用应用程序的身份或凭证来访问。

- kubernetes是一种管理容器化应用程序的平台。

- `kubectl`是一种用来和kubernetes交互的命令行工具。

- `curl`是一种用来发送HTTP请求的命令行工具。

- `kubectl proxy`可以创建一个本地代理，让我们可以访问kubernetes的API服务器。

- `-v=9`是一种用来显示详细信息的选项。

```bash
kubectl get nodes
```

在上面命令的输出结果中，我们可以找到对应的curl请求信息。

```bash
curl -v -XGET  \
  -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" \
  -H "User-Agent: kubectl/v1.24.1 (linux/amd64) kubernetes/3ddd0f4" \
  'https://<control_plane_ip>/api/v1/nodes?limit=500'
```

参考信息：

- [forum-like page](https://discuss.kubernetes.io/t/kubectl-tips-and-tricks/) 是有K8s运营的平台，提供了很多关于如何使用`kubectl`的详细信息和例子。

- [Manage multiple clusters and multiple config files](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

- [kubectl command documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

- [Shell autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)

- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

- [jsonpath in kubectl](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

- [kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
