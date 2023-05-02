# CKA自学笔记22:Policy

## ResourceQuota

演示场景：

* 为namespace `quota-object-example` 创建资源配额ResourceQuota `object-quota-demo`。
* 为 NodePort 测试 ResourceQuota `object-quota-demo`。
* 为 PVC 测试 ResourceQuota `object-quota-demo`。

### Create Namespace

创建Namespace。

```bash
kubectl create ns quota-object-example
```

### 创建ResourceQuota

为namespace `quota-object-example` 创建资源配额ResourceQuota `object-quota-demo`。
在该namespace中，我们只能创建 1 个永久卷PVC、1 个负载均衡LoadBalancer服务，不能创建 NodePort 服务。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota-demo
  namespace: quota-object-example
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
EOF
```

### 检查配额

```bash
kubectl get resourcequota object-quota-demo --namespace=quota-object-example --output=yaml
```

主要信息摘录如下：

```yaml
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
status:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
  used:
    persistentvolumeclaims: "0"
    services.loadbalancers: "0"
    services.nodeports: "0"
```

### 针对NodePort的额度测试

在namespace `quota-object-example` 中创建一个 Deployment `ns-quota-test`。

```bash
kubectl create deployment ns-quota-test --image nginx --namespace=quota-object-example
```

通过 NodePort 将 Deployment 暴露出来。

```bash
kubectl expose deployment ns-quota-test --port=80 --type=NodePort --namespace=quota-object-example
```

没有意外，我们得到下面的错误，因为我们设置了 Quota `services.nodeports: 0`。

```console
Error from server (Forbidden): services "ns-quota-test" is forbidden: exceeded quota: object-quota-demo, requested: services.nodeports=1, used: services.nodeports=0, limited: services.nodeports=0
```
  
### 针对PVC的额度测试

在namespace `quota-object-example` 中创建一个 PVC `pvc-quota-demo`。

```bash
kubectl applly -f - << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-quota-demo
  namespace: quota-object-example
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
EOF
```

检查配额信息。

```bash
kubectl get resourcequota object-quota-demo --namespace=quota-object-example --output=yaml
```

这里设置了 `persistentvolumeclaims` 为 `1`，并且配额也是 `1`。如果我们再次创建 PVC，则会收到 403 错误。

```yaml
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
status:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
  used:
    persistentvolumeclaims: "1"
    services.loadbalancers: "0"
    services.nodeports: "0"
```

## LimitRange

演示场景：

* Create LimitRange `cpu-limit-range` to define range of CPU Request and CPU Limit for a Container.
* Test LimitRange `cpu-limit-range` via Pod.
  * Scenario 1: Pod without specified limits
  * Scenario 2: Pod with CPU limit, without CPU Request
  * Scenario 3: Pod with CPU Request onlyl, without CPU Limits

* 创建 `cpu-limit-range` 限制范围，用于设定容器 CPU 请求和 CPU 限制的范围。
* 通过 Pod 测试 `cpu-limit-range` 限制范围。
  * 方案1：没有限制的 Pod
  * 方案2：只有 CPU 限制，但没有 CPU 请求限制的 Pod
  * 方案3：只有 CPU 请求限制，但没有 CPU 限制的 Pod

演示背景：

*LimitRange* 提供了以下约束：

* 强制限制 namespace 内每个 Pod 或容器的最小和最大计算资源使用情况。
* 强制限制 namespace 内每个 PersistentVolumeClaim 的最小和最大存储请求。
* 在 namespace 内为某个资源的请求和限制之间设置比率。
* 在 namespace 中设置默认的计算资源请求/限制，并在运行时自动将它们注入到容器中。

### 设置LimitRange

创建namespace `default-cpu-example` 用于下面的演示。

```bash
kubectl create namespace default-cpu-example
```

创建 *LimitRange* `cpu-limit-range`，用于定义容器的 CPU 请求和 CPU 限制范围。
应用此 *LimitRange* 资源后，CPU 限制将影响所有新创建的 Pod。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
  namespace: default-cpu-example
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
EOF
```

### 通过Pod进行测试

* 场景1: 没有限制的pod。

创建一个没有限制的pod。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo
  namespace: default-cpu-example
spec:
  containers:
  - name: default-cpu-demo-ctr
    image: nginx
EOF
```

验证我们创建的Pod的详细信息。Pod将从namespace中继承CPU Limit和CPU Request作为其默认值。

```bash
kubectl get pod default-cpu-demo --output=yaml --namespace=default-cpu-example
```

输出yaml文件的部分内容：

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: default-cpu-demo-ctr
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: 500m
```

* 场景2: 只有 CPU 限制，但没有 CPU 请求限制的 Pod

创建一个带有 CPU 限制但没有 CPU 请求的 Pod。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-limit
  namespace: default-cpu-example
spec:
  containers:
  - name: default-cpu-demo-limit-ctr
    image: nginx
    resources:
      limits:
        cpu: "1"
EOF

kubectl apply -f default-cpu-demo-limit.yaml
```

验证我们创建的 Pod 的细节。Pod 从namespace中继承 CPU 请求作为其默认值，并指定自己的 CPU 限制。

```bash
kubectl get pod default-cpu-demo-limit --output=yaml --namespace=default-cpu-example
```

输出的yaml文件部分内容：

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: default-cpu-demo-limit-ctr
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "1"
```

* 场景3: 只有 CPU 请求限制，但没有 CPU 限制的 Pod

创建一个只有 CPU 请求限制，但没有 CPU 限制的 Pod

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-request
  namespace: default-cpu-example
spec:
  containers:
  - name: default-cpu-demo-request-ctr
    image: nginx
    resources:
      requests:
        cpu: "0.75"
EOF
```

验证我们创建的 Pod 的详细信息。Pod 从命名空间继承了 CPU Limits 作为其默认值，并指定了自己的 CPU Requests。

```bash
kubectl get pod default-cpu-demo-request --output=yaml --namespace=default-cpu-example
```

输出的yaml文件部分内容：

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: default-cpu-demo-request-ctr
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: 750m
```
