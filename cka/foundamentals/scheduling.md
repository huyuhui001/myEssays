# CKA自学笔记20:Scheduling

演示场景：

* 为 Pod 配置 `nodeSelector`。
* 为 Node 配置 `nodeName`。
* 使用 `podAffinity` 来分组 Pod。

* 污点Taints和容忍Tolerations
  * 设置污点Taint
  * 设置容忍Toleration
  * 移除污点Taint

## nodeSelector

Let's assume the scenario below.

* We have a group of high performance servers.
* Some applications require high performance computing.
* These applicaiton need to be scheduled and running on those high performance servers.

We can leverage Kubernetes attributes node `label` and `nodeSelector` to group resources as a whole for scheduling to meet above requirement.

假设以下场景：

* 我们有一组高性能服务器。
* 一些应用需要高性能计算。
* 这些应用需要被调度并在高性能服务器上运行。

我们可以利用 Kubernetes 的 `label` 和 `nodeSelector` 属性来将资源作为一个整体进行分组，以满足上面的需求。

1.给节点设标签

给节点 `cka002` 设定标签值 `Configuration=hight`。

```bash
kubectl label node cka002 configuration=hight
```

执行下面的命令进行验证，我们会看看到节点`cka002`上有了标签信息`configuration=hight`。

```bash
kubectl get node --show-labels
```

2.给pod配置nodeSelector

创建一个 Pod 并使用 `nodeSelector` 将该 Pod 调度到指定的节点上。

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql-nodeselector
spec:
 selector:
   matchLabels:
     app: mysql
 template:
   metadata:
     labels:
       app: mysql
   spec:
     containers:
     - image: mysql:8.0
       name: mysql
       env:
       - name: MYSQL_ROOT_PASSWORD
         value: password
       ports:
       - containerPort: 3306
         name: mysql
     nodeSelector:
       configuration: hight
EOF
```

检查pod `mysql-nodeselector` 运行状态。

```bash
kubectl get pod -l app=mysql -o wide |  grep mysql-nodeselector
```

下面的执行结果说明pod `mysql-nodeselector` 正运行在节点 `cka002` 上。

```console
NAME                                  READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
mysql-nodeselector-6b7d9c875d-vs8mk   1/1     Running   0          7s     10.244.112.29   cka002   <none>           <none>
```

## nodeName

注意，`nodeName` 具有最高优先级，因为它不是由 `Scheduler` 进行调度的。

创建一个 Pod `nginx-nodename`，并将其指定在 `cka003` 节点上。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nodename
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: cka003
EOF
```

验证pod `nginx-nodename` 是否运行在节点 `ckc003` 上。

```bash
kubectl get pod -owide |grep nginx-nodename
```

运行结果：

```console
NAME                                      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
nginx-nodename                            1/1     Running   0          8s     10.244.102.29   cka003   <none>           <none>
```

## Affinity

在 Kubernetes 集群中，有些 Pod 需要频繁与其他 Pod 进行交互。在这种情况下，建议将这些 Pod 调度到同一个节点上运行。例如，两个 Pod：`Nginx` 和 `Mysql`，如果它们频繁通信，则需要在同一个节点上部署它们。

基于pod之间的关系，我们可以使用 `podAffinity` 进行选择。

`podAffinity` 有两种调度类型：

* `requiredDuringSchedulingIgnoredDuringExecution`（硬亲和）
* `preferredDuringSchedulingIgnoredDuringExecution`（软亲和）

可以使用以下类型设置 `topologyKey`：

* `kubernetes.io/hostname` ＃NodeName
* `failure-domain.beta.kubernetes.io/zone` ＃区域 Zone
* `failure-domain.beta.kubernetes.io/region` # 区域 Zone

我们可以设置节点标签来对节点的名称/区域进行分类，这样就可以被 `podAffinity` 所使用。

创建pod `Nginx`。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

创建pod `MySql`。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql
    env:
     - name: "MYSQL_ROOT_PASSWORD"
       value: "123456"
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        topologyKey: kubernetes.io/hostname
EOF
```

由于我们配置了 `podAffinity`，因此 Pod `mysql` 将会被调度到和 Pod `nginx` 相同的节点上，标签为 `app:nginx`。

通过命令 `kubectl get pod -o wide` 我们可以看到这两个 Pod 正在运行在节点 `cka002` 上。

## Taints & Tolerations

### 概念

节点的affinity是Pod的属性，可以将它们吸引到一组节点中（作为首选项或硬性要求）。Taints的作用相反——它们允许节点排斥一组Pod。

Tolerations应用于Pod。Tolerations允许调度器安排具有匹配Taints的Pod，但并不保证一定能够调度：调度器还会评估其他参数作为其功能的一部分。

Taints和tolerations共同确保Pod不会被调度到不合适的节点上。一个或多个Taints会应用于节点；这标志着该节点不应接受不容忍这些Taints的Pod。

在生产环境中，我们通常为控制平面节点配置Taints（实际上，大多数K8s安装工具会自动向控制平面节点添加Taints），因为控制平面仅运行Kubernetes系统组件。如果将其用于运行应用程序Pod，很容易消耗资源。最终，控制平面节点将崩溃。如果我们需要稍后将其他系统组件添加到控制平面节点，则可以为相应的系统组件Pod配置Tolerations以容忍Taints。

### 设置Taints

将 `cka003` 节点设置为 Taint 节点。将状态设置为 `NoSchedule`，这不会影响已在 `cka003` 上运行的现有 Pod。

```bash
kubectl taint nodes cka003 key=value:NoSchedule
```

### 设置Tolerations

我们可以使用 Tolerations 让 Pod 调度到一个 Taint 节点上。

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql-tolerations
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
      tolerations:
      - key: "key"
        operator: "Equal"
        value: "value"
        effect: "NoSchedule"
EOF
```

Deployment `mysql-tolerations`的Pod使用了tolerations设置，并被调度到了节点 `cka003` 上，而该节点是一个被污点化taint的节点。

```bash
kubectl get pod -o wide | grep mysql-tolerations
```

### 取消Taints

```bash
kubectl taint nodes cka003 key-
```
