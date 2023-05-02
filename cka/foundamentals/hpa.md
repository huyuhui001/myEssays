# CKA自学笔记21:Horizontal Pod Autoscaling (HPA)

演示场景：

* 安装 Metrics Server 组件
* 创建 Deployment `podinfo` 和 Service `podinfo` 用于压力测试
* 创建 HPA `my-hpa`
* 进行压力测试

## 安装Metrics Server

下载`components.yaml`文件，来部署Metrics Server。

```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

把yaml文件中google的镜像源替换为阿里的镜像源 `image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.1`。

```bash
sed -i 's/k8s\.gcr\.io\/metrics-server\/metrics-server\:v0\.6\.1/registry\.aliyuncs\.com\/google_containers\/metrics-server\:v0\.6\.1/g' components.yaml
```

修改deployment `metrics-server`的 `args`，添加选项 `--kubelet-insecure-tls` 以禁用证书验证。

```bash
vi components.yaml
```

更新 `args`。

```yaml
......
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.1
......
```

应用文件`components.yaml`来部署`metrics-server`。

```bash
kubectl apply -f components.yaml
```

下面是运行结果，相关资源被创建。

```console
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

验证 pod `metrics-server` 是否按预期在正常运行。

```bash
kubectl get pod -n kube-system -owide | grep metrics-server
```

运行结果。关注READY下的状态：`1/1` running代表正常运行。

```console
NAME                                       READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
metrics-server-7fd564dc66-sdhdc            1/1     Running   0          61s     10.244.102.15    cka003   <none>           <none>
```

查询每个节点上当前CPU和内存的用量情况。

```bash
kubectl top node
```

运行结果：

```console
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
cka001   595m         29%    1937Mi          50%       
cka002   75m          3%     1081Mi          28%       
cka003   79m          3%     1026Mi          26% 
```

## 部署服务`podinfo`

创建 Deployment `podinfo` 和 Service `podinfo` ，后面会进行更进一步的压力测试。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  type: NodePort
  ports:
    - port: 9898
      targetPort: 9898
      nodePort: 31198
      protocol: TCP
  selector:
    app: podinfo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfod
        image: stefanprodan/podinfo:0.0.1
        imagePullPolicy: Always
        command:
          - ./podinfo
          - -port=9898
          - -logtostderr=true
          - -v=2
        ports:
        - containerPort: 9898
          protocol: TCP
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "256Mi"
            cpu: "100m"
EOF
```

## Config HPA

创建一个名为 `my-hpa` 的 HPA，并将其绑定到名为 `podinfo` 的部署中，设定其 CPU 利用率为 `50%` 作为触发自动缩放的阈值，最小副本数为 `2`，最大副本数为 `10`。

使用以下命令创建 `my-hpa` HPA：

```bash
kubectl autoscale deployment podinfo --cpu-percent=50 --min=2 --max=10 --name=my-hpa
kubectl autoscale deployment podinfo --cpu-percent=50 --min=1 --max=10
```

使用 `autoscaling/v1` 版本的模版来创建HPA `my-hpa`。

```bash
kubectl apply -f - <<EOF
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
EOF
```

使用 `autoscaling/v2` 版本的模版来创建HPA `my-hpa`，在matrics中增加内存控制。

```bash
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 100Mi
EOF
```

查看HPA的状态。

```bash
kubectl get hpa
```

查询结果：

```console
NAME     REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
my-hpa   Deployment/podinfo   2%/50%    2         10        2          60s
```

提示：

* `metrics.resource` 表示要进行自动扩缩容的指标。在与目标值比较之前，这些指标值将被平均。
* `metrics.resource.target.type` 表示指标类型是 Utilization、Value 还是 AverageValue。
* `metrics.resource.target.averageUtilization` 是所有相关 Pod 资源指标平均值的目标值，表示为 Pod 请求该资源的百分比。当前仅适用于 Resource 指标类型。
* `metrics.resource.target.averageValue` (Quantity) 是所有相关 Pod 指标平均值的目标值，以数量形式表示。
* `metrics.resource.target.value` (Quantity) 是指标的目标值，以数量形式表示。

参考：

* [Algorithm details](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details)

## 压力测试

### 安装ab

这里我们使用 `ab` 工具模拟 1000 个并发请求。

`ab` 命令是一个命令行负载测试和基准测试工具，用于模拟向网站发送高流量。
Apache.org 中对`ab`的简短定义为：`ab` 首字母缩写代表 Apache Bench，其中 bench 是 benchmarking 的简写。

执行下面的命令安装 `ab` 工具。

```bash
sudo apt install apache2-utils -y
```

命令 `ab` 中最常用的2个选项是 `-n` 和 `-c`。

* `-n requests`：请求次数
* `-c concurrency`：并发数
* `-t timelimit`：压测时间，单位是秒，默认是50000
* `-p postfile`：POST 文件，同时也需要使用选项-T
* `-T content-type`：指定 Content-type。Content-type header是POST或PUT数据时用来指定数据类型的header，例如'application/x-www-form-urlencoded'和'text/plain'。默认为'text/plain'。
* `-k`：开启 HTTP KeepAlive 特性

例如：

```bash
ab -n 1000 -c 100 http://www.baidu.com/
```

### 并发压力测试

对当前节点运行命令 `ab` 进行1000并发请求模拟。节点端口 `31198` 是 `podinfo` 服务的端口。

```bash
ab -c 1000 -t 60 http://127.0.0.1:31198/
```

通过命令 `kubectl get hpa -w`，我们可以看到 CPU 工作负载一直在增加。

```console
NAME    REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
......
nginx   Deployment/podinfo   199%/50%    2         10        10         14m
nginx   Deployment/podinfo   934%/50%    2         10        10         14m
nginx   Deployment/podinfo   964%/50%    2         10        10         14m
nginx   Deployment/podinfo   992%/50%    2         10        10         15m
nginx   Deployment/podinfo   728%/50%    2         10        10         15m
nginx   Deployment/podinfo   119%/50%    2         10        10         15m
......
```

通过下面命令我们可以看到 Deployment `podinfo` 被触发了自动扩缩容。

```bash
kubectl get pod
kubectl get deployment
```

注意，扩容是一个逐步的过程，而不是一个突然的事件来达到最大值。当 CPU 工作负载下降时，它将被缩小到一个平衡的状态。

```bash
kubectl get hpa -w
```

几小时后，我们可以看到deployment的缩容结果。

```console
NAME     REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
my-hpa   Deployment/podinfo   2%/50%    2         10        2          60s
```

删除演示中所创建的临时资源。

```bash
kubectl delete service podinfo
kubectl delete deployment podinfo
kubectl delete hpa my-hpa
```
