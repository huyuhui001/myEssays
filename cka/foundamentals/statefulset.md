# CKA自学笔记12:StatefulSet

## 演示场景

* 创建一个 Headless Service `nginx` 和一个StatefulSet `web`
* 扩展 StatefulSet `web`

## 演示

创建一个 Headless Service `nginx` 和一个StatefulSet `web`

```bash
kubectl apply -f - << EOF
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
EOF
```

读取上一步创建的StatefulSet Pod 的详细信息。

```bash
kubectl get pod | grep web
```

运行结果

```bash
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          27s
web-1   1/1     Running   0          10s
```

使用命令 `kubectl edit sts web` 更新现有的 StatefulSet。
只有以下字段可以更新：`replicas`、`image`、`rolling updates`、`labels`、`resource request/limit` 和 `annotations`。

注意：当 StatefulSet Pod 在当前节点中死亡时，不会自动在其他节点中创建副本。

扩展 StatefulSet。
将 StatefulSet `web` 的副本数扩展到 `5`。

```bash
kubectl scale sts web --replicas=5
```

参考：

Partition表示在更新期间应将 StatefulSet 划分为哪个序号。
在滚动更新期间，从序号 Replicas-1 到 Partition 的所有 Pod 都会更新。
从序号 Partition-1 到 0 的所有 Pod 都保持不变。这对于进行金丝雀部署非常有用。默认值为0。

命令：`kubectl explain statefulsets.spec.updateStrategy.rollingUpdate.partition`

金丝雀部署是一种软件部署技术，其中在将新功能或版本发布给更大的用户子集或所有用户之前，先将其发布给生产中的一小部分用户。
这种技术是低风险的，因为新功能最初只部署给少量用户。
"Canary"一词源自旧的煤矿技术，当时金丝雀被用作空气中毒素的早期探测器。
在金丝雀部署中，目标环境中的所有基础设施都会以小阶段进行更新。
它用于测试新功能和升级以查看它们如何处理生产环境。

删除所创建的资源。

```bash
kubectl delete sts web
kubectl delete service nginx
```
