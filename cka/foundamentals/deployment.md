# CKA自学笔记9:Deployment

## 摘要

修改已有的Deployment，比如，增加端口号等。

## 演示

创建Deployment `nginx`。

```bash
kubectl create deployment nginx --image=nginx
```

执行以下命令以获取带有端口号的yaml模板。
选项 `--port=8080` 指定了该容器暴露的端口号。

```bash
kubectl create deployment nginx --image=nginx --port=8080 --dry-run=client -o yaml
```

这样我们就知道了添加端口号的路径，就像下面这样：

```bash
kubectl explain deployment.spec.template.spec.containers.ports.containerPort
```

执行下面的命令来修改当前正在运行的Deployment。

```bash
kubectl edit deployment nginx
```

添加下面2行来制定`8080`端口和`TCP`协议。

```yaml
spec:
  template:
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 8080
          protocol: TCP
```

通过命令 `kubectl describe deployment <deployment_name>`我们可以看到在Deployment中端口号和协议已经被正确添加了。

```yaml
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
```

通过命令 `kubectl describe pod <pod_name>` 我们可以看到在pod中端口号和协议已经被正确添加了。

```yaml
Containers:
  nginx:
    Container ID:   containerd://af4a1243f981497074b5c006ac55fcf795688399871d1dfe91a095321f5c91aa
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:1761fb5661e4d77e107427d8012ad3a5955007d997e0f4a3d41acc9ff20467c7
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 24 Jul 2022 22:50:12 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hftdt (ro)
```

以下是Deployment的一些关键字段（使用 `kubectl explain` ）：

* `deployment.spec.revisionHistoryLimit`：保留旧的`ReplicaSets`的数量，以便进行回滚。默认为 `10`。
* `deployment.spec.strategy.type`：部署的类型。可以是 `Recreate` 或 `RollingUpdate`。默认为 `RollingUpdate`。
* `deployment.spec.strategy.rollingUpdate.maxUnavailable`：在更新期间可以不可用的Pod的最大数量。默认为`25％`。
* `deployment.spec.strategy.rollingUpdate.maxSurge`：可以安排的Pod数量超出所需Pod数量的最大值。默认为`25％`。如果 `MaxUnavailable` 为 `0`，则此值不能为 `0`。
* `deployment.spec.minReadySeconds`：新创建的Pod的最小准备时间（所有容器都没有崩溃），以便被视为可用。默认为`0`（一旦准备好就会被视为可用）。
