# CKA自学笔记11:Namespace

## 演示场景

* 获取namespace列表
* 创建新的namespace
* 给namespace设定标签
* 删除一个namespace

## 演示

获取当前namespace列表。

```bash
kubectl get namespace
```

获取当前namespace列表和对应标签信息。

```bash
kubectl get ns --show-labels
```

创建一个namespace `cka`。

```bash
kubectl create namespace cka
```

给新创建的namespace `cka`设定标签。

```bash
kubectl label ns cka cka=true
```

在namespace `cka` 上创建 Nginx Deployment。

```bash
kubectl create deploy nginx --image=nginx --namespace cka
```

在namespace `cka`上检查正在运行的deployment和pod。

```bash
kubectl get deploy,pod -n cka
```

运行结果：

```console
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           2m14s

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-85b98978db-bmkhf   1/1     Running   0          2m14s
```

删除namespace `cka`，则所有运行在这个namespace上的资源都会被删除。

```bash
kubectl delete ns cka
```

如果在删除某个namespace时遇到状态一直是`Terminating`，则可以尝试用下面的方法解决。

```bash
kubectl get namespace $NAMESPACE -o json | sed -e 's/"kubernetes"//' | kubectl replace --raw "/api/v1/namespaces$NAMESPACE/finalize" -f -
```
