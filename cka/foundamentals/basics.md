# kubectl basics

!!! Scenario
    Get to know how to operate Kubernetes cluster using `kubectl`. 
    
    * via [API](https://kubernetes.io/docs/reference/kubernetes-api/)
    * via kubectl
    * via Dashboard


Demo: 

## Check current kubeconfig file.

Use the `kubectl config` command to get current context of configuration file.
```console
echo $KUBECONFIG
kubectl config view
kubectl config get-contexts
```

## Get resource list

Get a complete list of supported resources
```console
kubectl api-resources
```

## Get cluster status.

Kubernetes control plane is running at `https://<control_plane_ip>:6443`

CoreDNS is running at `https://<control_plane_ip>:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy`
```console
kubectl cluster-info
kubectl cluster-info dump
```

## Display resources.

Use `kubectl get --help` to get examples of displaying one or many resources.

Get health status of control plane.
```console
kubectl get componentstatuses
kubectl get cs
```
Result
```
NAME                 STATUS    MESSAGE                         ERROR
etcd-0               Healthy   {"health":"true","reason":""}   
scheduler            Healthy   ok                              
controller-manager   Healthy   ok 
```

## Get node status and details

```console
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node cka001
```

Use command `kubectl create --help` to get examples of creating resources.

## Create namespace

```console
kubectl create namespace --help
kubectl create namespace my-namespace
```

!!! information
    Namespace is a cluster, which includes services. Service may be on a node, may be not. 


## Create deployment

Create Deployment on the namespace.
```console
kubectl -n my-namespace create deployment my-busybox \
  --image=busybox \
  --replicas=3 \
  --port=5701
```

## Create ClusterRole

```console
kubectl create clusterrole --help

kubectl create clusterrole pod-creater \
  -n my-namespace \
  --verb=create \
  --resource=deployment \
  --resource-name=my-busybox
```

## Create ServiceAccount

```console
kubectl create serviceaccount --help
kubectl -n my-namespace create serviceaccount my-service-account
```

## Create RoleBinding

!!! Note
    `RoleBinding` can reference a Role in the same namespace or a ClusterRole in the global namespace.

```console
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

## Use the proxy

We can use `kubectl proxy` command to open a tunnel to the API server and make it available locally - usually on localhost:8001 / 127.0.0.1:8001. 
When I want to explore the API, this is an easy way to gain access.

Run the command `kubectl proxy &` and open `http://localhost:8001/api/v1` in browser.
Just opening `http://localhost:8001` will return an error because we are only allowed to access certain parts of the API. Hence the API path is important
```
kubectl proxy &
```
Output
```
[1] 102358
Starting to serve on 127.0.0.1:8001
```

Example, get available API groups and so on via below link:
```
http://127.0.0.1:8001/
http://127.0.0.1:8001/api/v1
http://127.0.0.1:8001/api/v1/namespaces
http://127.0.0.1:8001/api/v1/namespaces/default
http://127.0.0.1:8001/api/v1/namespaces/sock-shop/pods
```


## Access as application

If we access kubernetes as an application rather than an administrator, we cannot use the `kubectl`. Instead of `kubectl` we can use the program `curl`.
We have to send HTTP requests to the cluster. asking for the available nodes.

Make sure `kubectl proxy` is running and serving on `http://localhost:8001/`.

Execute command below with a `-v=9` flag, it shows all the information needed.
```
kubectl get nodes
```

Go through the command's output and find the correct curl request below.
```console
curl -v -XGET  \
  -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" \
  -H "User-Agent: kubectl/v1.24.1 (linux/amd64) kubernetes/3ddd0f4" \
  'https://<control_plane_ip>/api/v1/nodes?limit=500'
```



!!! Reference
    * There is a [forum-like page](https://discuss.kubernetes.io/t/kubectl-tips-and-tricks/) hosted by K8s with lots of information around kubectl and how to use it best.
    * [Manage multiple clusters and multiple config files](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
    * [kubectl command documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
    * [Shell autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)
    * [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
    * [jsonpath in kubectl](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
    * [kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)







