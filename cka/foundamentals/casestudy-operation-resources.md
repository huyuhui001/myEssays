# 主题讨论:Kubernetes资源常见操作

演示场景：

* 节点标签（Node Label）
* 注解（Annotation）
* 命名空间（Namespace）
* ServiceAccount 授权（ServiceAccount Authorization）
  * 授权默认 ServiceAccount 访问 API
* 部署（Deployment）
* 暴露服务（Expose Service）
* 扩展部署（Scale out the Deployment）
* 滚动升级（Rolling update）
* 回滚升级（Rolling back update）
* 事件（Event）
* 日志记录（Logging）

## 节点标签（Node Label）

* 添加/修改/移出节点标签。

```bash
# Update node label
kubectl label node cka002 node=demonode

# Get node info with label info
kubectl get node --show-labels

# Search node by label
kubectl get node -l node=demonode

# Remove a lable of node
kubectl label node cka002 node-
```

## 注解（Annotation）

创建deployment `Nginx`。

```bash
kubectl create deploy nginx --image=nginx:mainline
```

获取注解信息

```bash
kubectl describe deployment/nginx
```

运行结果：

```yaml
......
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
......
```

添加新的注解信息。

```bash
kubectl annotate deployment nginx owner=James.H
```

再次查询注解信息得到如下结果。

```yaml
......
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
                        owner: James.H
Selector:               app=nginx
......
```

更新/覆盖注解信息。

```bash
kubectl annotate deployment/nginx owner=K8s --overwrite
```

再次查询注解信息得到如下结果。

```yaml
......
Annotations:            deployment.kubernetes.io/revision: 1
                        owner: K8s
Selector:               app=nginx
......
```

移除注解信息。

```bash
kubectl annotate deployment/nginx owner-
```

运行结果：

```yaml
......
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
......
```

删除上面演示中创建的临时资源。

```bash
kubectl delete deployment nginx
```

## 命名空间（Namespace）

* 查询当前可用namespace。

```bash
kubectl get namespace
```

运行结果：

```console
NAME              STATUS   AGE
default           Active   3h45m
dev               Active   3h11m
kube-node-lease   Active   3h45m
kube-public       Active   3h45m
kube-system       Active   3h45m
```

* 查询某个namespace上运行的pod信息。

```bash
kubectl get pod -n kube-system
```

运行结果：

```console
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5c64b68895-jr4nl   1/1     Running   0          3h25m
calico-node-dsx76                          1/1     Running   0          3h25m
calico-node-p5rf2                          1/1     Running   0          3h25m
calico-node-tr22l                          1/1     Running   0          3h25m
coredns-6d8c4cb4d-g4jxc                    1/1     Running   0          3h45m
coredns-6d8c4cb4d-sqcvj                    1/1     Running   0          3h45m
etcd-cka001                                1/1     Running   0          3h45m
kube-apiserver-cka001                      1/1     Running   0          3h45m
kube-controller-manager-cka001             1/1     Running   0          3h45m
kube-proxy-5cdbj                           1/1     Running   0          3h41m
kube-proxy-cm4hc                           1/1     Running   0          3h45m
kube-proxy-g4w52                           1/1     Running   0          3h41m
kube-scheduler-cka001                      1/1     Running   0          3h45m
```

* 查询所有namespace上的pod信息。

```bash
kubectl get pod --all-namespaces
kubectl get pod -A
```

## ServiceAccount 授权（ServiceAccount Authorization）

在 Kubernetes 1.23 及更低版本中，当我们创建一个新的命名空间时，Kubernetes 会自动创建一个名为 `default` 的 ServiceAccount 和一个名为 `default-token-xxxxx` 的令牌。

而在 Kubernetes 1.24 中，创建新的命名空间时仅会自动创建一个名为 `default` 的 ServiceAccount，需要手动创建与 `default` ServiceAccount 相关联的令牌。

以下是创建一个名为 `dev` 的新命名空间的示例，我们可以看到在命名空间 `dev` 中仅创建了 ServiceAccount：`default`，没有与 ServiceAccount `default` 相关联的令牌（secret）。

```bash
kubectl create namespace dev
kubectl get serviceaccount -n dev
kubectl get secrets -n dev
```

有一个默认的集群角色 `admin`，但是没有将其绑定到任何集群角色绑定（clusterrole binding）中。

```bash
kubectl get clusterrole admin
kubectl get clusterrolebinding | grep ClusterRole/admin
```

角色Role和角色绑定RoleBinding是基于命名空间的。在命名空间`dev`中，没有角色和角色绑定。

```bash
kubectl get role -n dev
kubectl get rolebinding -n dev
```

在 Kubernetes 集群中，Secret 是一个对象，用于存储敏感信息，如用户名、密码和令牌等。Secret 的目标是对凭据进行编码或哈希化。这些凭据可以在各种 Pod 定义文件中重复使用。

`kubernetes.io/service-account-token` 类型的 Secret 用于存储标识服务账户的令牌。使用此类型的 Secret 时，需要确保 `kubernetes.io/service-account.name` 注释设置为现有服务账户名称。

让我们在 `dev` 命名空间中为 ServiceAccount `default` 创建一个令牌。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Secret
metadata:
  name: default-token-dev
  namespace: dev
  annotations:
    kubernetes.io/service-account.name: "default"
type: kubernetes.io/service-account-token

EOF
```

现在在 `dev` 命名空间中创建了 ServiceAccount `default` 和 Secret（令牌） `default-token-dev`。

```bash
kubectl get serviceaccount -n dev
kubectl get secrets -n dev
```

获取默认 Service Account 的 token，并赋值给环境变量`$TOKEN`。

```bash
TOKEN=$(kubectl -n dev describe secret $(kubectl -n dev get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d ' ')
echo $TOKEN
```

获取 API Service 地址，并赋值给环境变量`$APISERVER`。

```bash
APISERVER=$(kubectl config view | grep https | cut -f 2- -d ":" | tr -d " ")
echo $APISERVER
```

通过 API Server 以 JSON 格式获取命名空间 `dev` 中的 Pod 资源。

```bash
curl $APISERVER/api/v1/namespaces/dev/pods --header "Authorization: Bearer $TOKEN" --insecure
```

我们将收到“403 forbidden”的错误消息。ServiceAccount `default`没有访问名称空间`dev`中的Pod的授权。

让我们创建一个名为`rolebinding-admin`的RoleBinding，将集群角色`admin`绑定到名称空间`dev`中的ServiceAccount `default`。
因此，ServiceAccount `default`被授予在名称空间`dev`中的管理员授权。

```bash
# Usage:
kubectl create rolebinding <rule> --clusterrole=<clusterrule> --serviceaccount=<namespace>:<name> --namespace=<namespace>

# Crate rolebinding:
kubectl create rolebinding rolebinding-admin --clusterrole=admin --serviceaccount=dev:default --namespace=dev
```

执行命令 `kubectl get rolebinding -n dev`，得到类似如下的结果。

```console
NAME                ROLE                AGE
rolebinding-admin   ClusterRole/admin   10s
```

再次通过 API Server 以 JSON 格式获取命名空间 `dev` 中的 Pod 资源，成功。

```bash
curl $APISERVER/api/v1/namespaces/dev/pods --header "Authorization: Bearer $TOKEN" --insecure
```

删除上面演示中创建的临时资源。

```bash
kubectl delete namespace dev
```

## 部署（Deployment）

创建一个 Ubuntu Pod 以进行操作，并附加到运行中的 Pod。

```bash
kubectl create -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
  labels:
    app: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: ["/bin/sleep", "3650d"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF

kubectl exec --stdin --tty ubuntu -- /bin/bash
```

创建一个 Deployment，选项 `--image` 指定了一个镜像，选项 `--port` 指定了外部访问的端口。
在创建 Deployment 的同时也会创建一个 Pod。

```bash
kubectl create deployment myapp --image=docker.io/jocatalin/kubernetes-bootcamp:v1 --replicas=1 --port=8080
```

查询deployment的状态。

```bash
kubectl get deployment myapp -o wide
```

输出结果：

```console
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS            IMAGES                                       SELECTOR
myapp   1/1     1            1           79s   kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v1   app=myapp
```

查询deployment的详细信息。

```bash
kubectl describe deployment myapp
```

运行结果：

```console
Name:                   myapp
Namespace:              dev
CreationTimestamp:      Sat, 23 Jul 2022 14:36:43 +0800
Labels:                 app=myapp
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=myapp
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
  Containers:
   kubernetes-bootcamp:
    Image:        docker.io/jocatalin/kubernetes-bootcamp:v1
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   myapp-b5d775f5d (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  95s   deployment-controller  Scaled up replica set myapp-b5d775f5d to 1
```

## 暴露服务（Expose Service）

获取上面练习中创建的pod和deployment的信息。

```bash
kubectl get deployment myapp -o wide
kubectl get pod -o wide
```

运行结果：

```console
NAME                    READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
myapp-b5d775f5d-cx8dx   1/1     Running   0          2m34s   10.244.102.7   cka003   <none>           <none>
```

执行命令`curl 10.244.102.7:8080`，以发送HTTP请求到pod的端口，得到如下结果。

```console
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-6jtgs | v=1
```

要使 Pod 可以从外部访问，需要将端口 `8080` 暴露给节点端口（NodePort）。这需要创建一个相关的 Service。

```bash
kubectl expose deployment myapp --type=NodePort --port=8080
```

执行命令`kubectl get svc myapp -o wide`，获取service `myapp` 的详细信息。

```console
NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE   SELECTOR
myapp   NodePort   11.244.74.3   <none>        8080:30514/TCP   7s    app=myapp
```

执行命令`curl 11.244.74.3:8080`，以发送HTTP请求到service的端口，得到如下结果。

```console
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-cx8dx | v=1
```

获取service的详细信息。

```bash
kubectl get svc myapp -o yaml
kubectl describe svc myapp
```

执行 `kubectl get endpoints myapp -o wide` 命令以获取相关的 `myapp` 端点（endpoint）的详细信息。

```console
NAME    ENDPOINTS           AGE
myapp   10.244.102.7:8080   43s
```

提示：

Endpoint（端点）是一个Kubernetes中的对象，用于存储可以被服务访问的Pod的网络地址和端口信息。
当Service创建时，Kubernetes会自动创建和更新相应的Endpoint。
Endpoint是由kube-proxy自动创建和维护的，并根据选择器匹配对应的Service和Pod。

能成功的发送HTTP请求到service和节点，说明pod的端口`8080`被正确的映射到节点的端口`32566`。

执行命令curl <cka003_node_ip>:30514，发送HTTP请求到节点`cka003`上对应的端口。

```console
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-6jtgs | v=1
```

登录进入Ubuntu pod，我们可以通过发送HTTP请求到`myapp`所映射的service、pod和节点的端口。

```bash
kubectl exec --stdin --tty ubuntu -- /bin/bash
curl 10.244.102.7:8080
curl 11.244.74.3:8080
curl <cka003_node_ip>:30514
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-6jtgs | v=1
```

## 扩容部署（Scale out the Deployment）

Scale out by replicaset. We set three replicasets to scale out deployment `myapp`. The number of deployment `myapp` is now three.
通过副本集replicaset进行扩展。我们通过指定副本集的方式，对deployment `myapp`进行扩展部署，下面例子中，deployment `myapp` 的副本数是三个。

```bash
kubectl scale deployment myapp --replicas=3
```

查询deployment的信息。

```bash
kubectl get deployment myapp
```

查询replicaset的信息

```bash
kubectl get replicaset
```

## 滚动升级（Rolling update）

命令用法：`kubectl set image (-f 文件名 | 类型 名称) 容器名称_1=容器镜像_1 ... 容器名称_N=容器镜像_N`。

使用命令 `kubectl get deployment`，我们可以获取deployment `myapp` 和相关容器 `kubernetes-bootcamp`。

```bash
kubectl get deployment myapp -o wide
```

使用命令`kubectl set image`来更新多个版本的镜像，并使用选项`--record`将更改记录在部署的注释中。

```bash
kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2 --record
```

查询当前replicas的状态。

```bash
kubectl get replicaset -o wide -l app=myapp
```

输出结果如下，pod正以新的副本集replicas数量运行。

```console
NAME               DESIRED   CURRENT   READY   AGE   CONTAINERS            IMAGES                                       SELECTOR
myapp-5dbd68cc99   1         1         0       8s    kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v2   app=myapp,pod-template-hash=5dbd68cc99
myapp-b5d775f5d    3         3         3       14m   kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v1   app=myapp,pod-template-hash=b5d775f5d
```

我们可以在对应的yaml文件中 `metadata.annotations` 部分获取变更历史记录。

```bash
kubectl get deployment myapp -o yaml
```

运行结果：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubernetes.io/change-cause: kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2
      --record=true
  ......
```

我们也可以使用命令 `kubectl rollout history` 获取更新历史记录，并使用特定修订版本号 `--revision=<revision_number>` 显示详细信息。

```bash
kubectl rollout history deployment/myapp
```

运行结果：

```yaml
deployment.apps/myapp 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2 --record=true
```

获取特定版本回滚历史记录。

```bash
kubectl rollout history deployment/myapp --revision=2
```

执行命令 `kubectl rollout undo` 可以回滚到上一个版本，或使用选项 `--to-revision=<revision_number>` 回滚到指定的版本。

```bash
kubectl rollout undo deployment/myapp --to-revision=1
```

版本 `1` 现在已经被替换成版本 `3`了。

```bash
kubectl rollout history deployment/myapp
```

运行结果：

```yaml
deployment.apps/myapp 
REVISION  CHANGE-CAUSE
2         kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2 --record=true
3         <none>
```

## 事件（Event）

获取指定pod的事件信息。

```bash
kubectl describe pod myapp-b5d775f5d-jlx6g
```

输出结果：

```console
......
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  54s   default-scheduler  Successfully assigned dev/myapp-b5d775f5d-jlx6g to cka003
  Normal  Pulled     53s   kubelet            Container image "docker.io/jocatalin/kubernetes-bootcamp:v1" already present on machine
  Normal  Created    53s   kubelet            Created container kubernetes-bootcamp
  Normal  Started    53s   kubelet            Started container kubernetes-bootcamp
```

查询集群的事件信息。

```bash
kubectl get event
```

## 日志记录（Logging）

查询pod的日志信息。

```bash
kubectl logs -f <pod_name>
kubectl logs -f <pod_name> -c <container_name> 
```

例如：

```console
kubectl logs -f myapp-b5d775f5d-jlx6g
```

运行结果：

```console
Kubernetes Bootcamp App Started At: 2022-07-23T06:54:18.532Z | Running On:  myapp-b5d775f5d-jlx6g
```

查询K8s不同组件的日志信息。

```bash
kubectl logs kube-apiserver-cka001 -n kube-system
kubectl logs kube-controller-manager-cka001 -n kube-system
kubectl logs kube-scheduler-cka001 -n kube-system
kubectl logs etcd-cka001 -n kube-system
systemctl status kubelet
journalctl -fu kubelet
kubectl logs kube-proxy-5cdbj -n kube-system
```

删除演示中创建的临时资源。

```bash
kubectl delete service myapp
kubectl delete deployment myapp
```
