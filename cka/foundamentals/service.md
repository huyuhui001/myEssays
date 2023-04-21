# CKA自学笔记10:Service

## 摘要

演示场景：

* 创建名为`httpd-app`的Deployment。
* 创建类型为 `ClusterIP` 的`httpd-app`服务，默认类型是 ClusterIP，只能内部访问。
* 验证对Pod的IP和服务的集群IP的访问。
* 将`httpd-app`服务类型更新为 `NodePort`，不需要对 `httpd-app`这个Deployment进行任何更改。
* 验证节点node的访问。对节点node对访问将被路由到Pod，从而实现从外部访问我们创建的服务`httpd-app`。
* 创建无头服务（Headless Service）`web` 和 有状态副本集（StatefulSet）`web`。
* 服务的内部流量策略。

`NodePort`可以翻译为“节点端口”，是一种Service的类型，可以通过在每个节点上打开一个端口，将Service暴露到集群外部。

`ClusterIP`可以翻译为“集群IP”，也是一种Service的类型，为Service提供了一个虚拟IP地址，可以在集群内部进行访问。这个IP地址通常由集群中的Kubernetes代理自动分配，并且只能在集群内部使用。

## ClusterIP

### 创建Service

创建Deployment `http-app`。

创建Service `httpd-app`并通过标签选择器（Label Selector）关联到Development `http-app`。
Service的类型是`ClusterIP`，这是Service的默认类型，只限于内部访问。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpd-app
spec:
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: httpd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-app
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 2
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80
EOF
```

执行命令 `kubectl get deployment,service,pod -o wide` 来查看使用上面yaml文件创建的Deployment和Service的状态。

```console
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/httpd-app   0/2     2            0           14s   httpd        httpd    app=httpd

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/httpd-app   ClusterIP   11.244.247.7   <none>        80/TCP    14s   app=httpd

NAME                             READY   STATUS    RESTARTS   AGE    IP              NODE     NOMINATED NODE   READINESS GATES
pod/httpd-app-6496d888c9-4nb6z   1/1     Running   0          77s    10.244.102.21   cka003   <none>           <none>
pod/httpd-app-6496d888c9-b7xht   1/1     Running   0          77s    10.244.112.19   cka002   <none>           <none>
```

执行下面的命令，验证对pod的IP地址的访问。

```bash
curl 10.244.102.21
curl 10.244.112.19
```

可以得到下面的信息，说明访问成功。

```console
<html><body><h1>It works!</h1></body></html>
```

执行下面的命令，验证通过端口对ClusterIP的访问。

```bash
curl 11.244.247.7:80
```

可以得到下面的信息，说明访问成功。

```console
<html><body><h1>It works!</h1></body></html>
```

### 暴露Service

创建一个临时的Pod `nslookup`，并附加到它以验证DNS解析。选项`--rm`表示在退出后删除该Pod。

```bash
kubectl run -it nslookup --rm --image=busybox:1.28
```

连接到这个Pod后，运行命令 `nslookup httpd-app`。我们会收到 `httpd-app` 服务的 ClusterIP 和完整的域名。

```console
/ # nslookup httpd-app
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      httpd-app
Address 1: 11.244.247.7 httpd-app.dev.svc.cluster.local
```

我们可以通过执行命令 `kubectl get pod -o wide` 来在新的终端中检查临时 Pod `nslookup` 的 IP 地址。Pod `nslookup` 的 IP 地址为 `10.244.112.20`。

```bash
kubectl get pod nslookup
```

运行结果

```console
NAME       READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
nslookup   1/1     Running   0          2m44s   10.244.112.20   cka002   <none>           <none>
```

## NodePort

创建并应用文件 `svc-nodeport.yaml` 来创建Service `httpd-app`。

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpd-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  selector:
     app: httpd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-app
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 2
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80
EOF
```

We will receive below output. The command `kubectl apply -f <yaml_file>` will update configuration to existing resources.
Here the Service `httpd-app` is changed from `ClusterIP` to `NodePort` type. No change to the Deployment `httpd-app`.

我们将收到以下输出。
其中，命令 `kubectl apply -f <yaml_file>` 将更新现有资源的配置。
在这里，Service `httpd-app` 从 `ClusterIP` 类型更改为 `NodePort` 类型。
对Deployment `httpd-app` 没有任何更改。

```console
service/httpd-app configured
deployment.apps/httpd-app unchanged
```

通过命令`kubectl get svc`来检查Service `httpd-app`，其中：

* Service的IP地址不变。
* Service的类型变为`NodePort`。
* Service的端口号从`80/TCP`变为`80:30080/TCP`。

```console
NAME        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
httpd-app   NodePort   11.244.247.7   <none>        80:30080/TCP   18m
```

在每个节点node上执行命令`curl <your_node_ip>:30080`，测试对Service `httpd-app`的联通性。

```bash
curl <node1_ip>:30080
curl <node2_ip>:30080
curl <node3_ip>:30080
```

可以得到下面的信息，说明访问成功。

```console
<html><body><h1>It works!</h1></body></html>
```

## Headless Service

创建Headless Service `web` 和StatefulSet `web`.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
          name: web
EOF
```

执行命令`kubectl get pod -owide -l app=web`检查刚才创建的pod。

```console
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
web-0   1/1     Running   0          9s    10.244.102.22   cka003   <none>           <none>
web-1   1/1     Running   0          6s    10.244.112.21   cka002   <none>           <none>
```

执行命令`kubectl describe svc -l app=web`，检查创建的Service的详细信息。

```yaml
Name:              web
Namespace:         dev
Labels:            app=web
Annotations:       <none>
Selector:          app=web
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              web  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.102.22:80,10.244.112.21:80
Session Affinity:  None
Events:            <none>
```

连接到临时Pod `nslookup`，通过 `nslookup` 来验证DNS到解析。

```bash
kubectl run -it nslookup --rm --image=busybox:1.28
```

通过 `nslookup` 命令访问Headless Service `web`，我们可以得到2个pod的IP地址，注意不是集群IP地址ClusterIP（因为是Headless Service）。

```console
/ # nslookup web
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web
Address 1: 10.244.112.21 web-1.web.dev.svc.cluster.local
Address 2: 10.244.102.22 web-0.web.dev.svc.cluster.local
```

我们也可以使用 `nslookup` 命令来查找 `web-0.web` 和 `web-1.web`。Headless Service的每个 Pod 都有自己的服务名称用于 DNS 查找。

```console
/ # nslookup web-0.web
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.web
Address 1: 10.244.102.22 web-0.web.dev.svc.cluster.local

/ # nslookup web-1.web
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.web
Address 1: 10.244.112.21 web-1.web.dev.svc.cluster.local
```

删除上面创建的临时资源。

```bash
kubectl delete sts web
kubectl delete service httpd-app web
kubectl delete deployment httpd-app 
```

## 服务内部流量策略

Service Internal Traffic Policy（服务内部流量策略）是Kubernetes中一种用于控制服务内部流量的策略。它的主要目的是控制Service对象中Pod的访问策略。

在Kubernetes中，Service对象将一个虚拟IP地址绑定到一组Pod上，以便可以通过这个虚拟IP地址来访问这组Pod。Service对象在某种程度上像负载均衡器，可以将请求流量路由到其下面的Pod。 Service对象通常会使用以下两种类型之一来路由流量：

* ClusterIP：此类型的Service只能从同一Kubernetes集群内的其他Pod访问，因为它是在Kubernetes集群内部路由请求流量的。
* NodePort：此类型的Service在所有节点上公开了一个静态端口，从而可以从集群外部访问它。但是，它仍然可以从集群内部访问。

Service Internal Traffic Policy定义了Pod如何访问同一个Service中的其他Pod。可以将其设置为以下三个选项之一：

* Cluster：这是默认设置，它允许Service中的任何Pod都可以访问另一个Pod。
* Local：此选项允许Pod仅访问在同一节点上运行的其他Pod。如果Pod需要快速的、低延迟的网络访问，可以使用此选项。
* Disabled：此选项将完全禁止Service内部的流量。它适用于特定的环境和部署中。

演示场景：

* 模拟 Service 内部流量策略的工作方式。
* 预期结果：
  * 通过设置 Service 的 `internalTrafficPolicy: Local`，Service 只会将流量路由到 Pod 所在的节点内部。

演示目的：

* Service Internal Traffic Policy（服务内部流量策略）可以限制内部流量仅路由到同一节点中的终端节点。
* 这里的“内部”流量是指源自当前集群中的Pod的流量。
* 通过将其 `.spec.internalTrafficPolicy` 设置为 Local，可以告诉 kube-proxy 仅对集群内部流量使用本地节点的终端节点。
* 对于位于没有给定服务的终端节点的节点上的Pod，即使服务在其他节点上有终端节点，该服务也会被视为在该节点上没有终端节点（对于该节点上的Pod）。

演示：

创建 Deployment `my-nginx` 和 Service `my-nginx`.

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
```

使用命令 `kubectl get pod -o wide`，我们可以得知 Deployment `my-nginx` 的 Pod 正在运行在 `cka003` 节点上。

```console
NAME                                      READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
my-nginx-cf54cdbf7-bscf8                  1/1     Running   0          9h      10.244.112.63   cka002   <none>           <none>
```

让我们从 `cka001` 发送 http 请求到运行在 `cka002` 上的 Pod。
我们将收到 `Welcome to nginx!` 信息，这意味着该 Pod 可以从其他节点访问。

```bash
curl 11.244.163.60
```

现在来修改Service `my-nginx`并指定`internalTrafficPolicy: Local`。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
  internalTrafficPolicy: Local
EOF
```

Let's send http request from `cka001` to the http request to the Pod again.
We will receive `curl: (7) Failed to connect to 11.244.163.60 port 80: Connection refused` error information.

我们再次从`cka001`发送http请求到该Pod。
我们将收到错误信息`curl: (7) Failed to connect to 11.244.163.60 port 80: Connection refused`。

```bash
curl 11.244.163.60
```

让我们登录到 `cka002` 节点并再次向 Pod 发送 HTTP 请求。
我们将收到 `Welcome to nginx!` 的信息。

```bash
curl 11.244.163.60
```

演示结论：

* 设置 Service 的 `internalTrafficPolicy: Local` 后，Service 只会将流量路由到当前 Pod 所在的节点内部的 Pod。

演示场景：

* 创建一个`nginx` Deployment
* 添加`nginx` Pod的端口号和别名
* 使用本地流量将Deployment暴露出去。

演示：

使用端口号为`80`创建`my-nginx` Deployment。

```bash
kubectl create deployment my-nginx --image=nginx --port=80
```

修改Deployment。

```bash
kubectl edit deployment my-nginx
```

在`my-nginx` Deployment中添加端口别名 `http`。

请参考以下部署的 YAML 模板链接：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>。

```yaml
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
          name: http
```

使用 `NodePort` 类型暴露 deployment。

```bash
kubectl expose deployment my-nginx --port=80 --target-port=http --name=my-nginx-svc --type=NodePort
```

修改service，把 `internalTrafficPolicy` 从 `Cluster` 改为 `Local`。

```bash
kubectl edit svc my-nginx-svc 
```

验证访问。
注意，Pod 正在运行在节点 cka003 上。我们将看到以下预期结果。

```bash
curl <deployment_pod_ip>:80    # succeed on node cka003. internalTrafficPolicy is effective.
curl <service_cluster_ip>:80   # succeed on all nodes.
curl <node_ip>:<ext_port>      # succeed on all nodes.
```
