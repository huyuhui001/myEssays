# CKA自学笔记25:Troubleshooting

## 事件

演示场景：

* 描述pod以获取事件信息。

演示：

命令用法：

```bash
kubectl describe <resource_type> <resource_name> --namespace=<namespace_name>
```

查询pod的事件信息。

创建一个Tomcat的pod。

```bash
kubectl run tomcat --image=tomcat
```

查询pod的事件信息。

```bash
kubectl describe pod/tomcat
```

得到类似下面的事件信息。

```console
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  55s   default-scheduler  Successfully assigned dev/tomcat to cka002
  Normal  Pulling    54s   kubelet            Pulling image "tomcat"
  Normal  Pulled     21s   kubelet            Successfully pulled image "tomcat" in 33.134162692s
  Normal  Created    19s   kubelet            Created container tomcat
  Normal  Started    19s   kubelet            Started container tomcat
```

查询namespace的事件信息。

```bash
kubectl get events -n <your_namespace_name>
```

得到类似下面的默认namespace的事件信息。

```console
LAST SEEN   TYPE      REASON           OBJECT                          MESSAGE
70s         Warning   FailedGetScale   horizontalpodautoscaler/nginx   deployments/scale.apps "podinfo" not found
2m16s       Normal    Scheduled        pod/tomcat                      Successfully assigned dev/tomcat to cka002
2m15s       Normal    Pulling          pod/tomcat                      Pulling image "tomcat"
102s        Normal    Pulled           pod/tomcat                      Successfully pulled image "tomcat" in 33.134162692s
100s        Normal    Created          pod/tomcat                      Created container tomcat
100s        Normal    Started          pod/tomcat                      Started container tomcat
```

得到类似下面的所有的namespace的事件信息。

```bash
kubectl get events -A
```

## 日志

演示场景：

* 查询pod的日志

命令用法：

```bash
kubectl logs <pod_name> -n <namespace_name>
```

选项：

* `--tail <n>`: 显示输出的最近 `<n>` 行。
* `-f`：实时流式显示输出。

显示输出的最近100行输出。

```bash
kubectl logs -f tomcat --tail 100
```

如果是一个多容器pod，则使用选项`-c`来指定某个特定的容器。

```bash
kubectl logs -f tomcat --tail 100 -c tomcat
```

## 节点可用性

### 查看可用节点

演示场景：

* 查看节点可用性

演示：

方式1：

```bash
kubectl describe node | grep -i taint
```

手工方式检查日志，下面的例子说明2个节点处于不可用状态。

```yaml
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             <none>
Taints:             <none>
```

方式2：

```bash
kubectl describe node | grep -i taint |grep -vc NoSchedule
```

这里我们会得到相同的结果，2个节点处于不可用状态。这里的`-v`表示排除，`-c`表示计数。

### 查看不可用节点

演示场景：

当我们在Worker节点 `cka002` 上停止 `kubelet` 服务时，

* 每个节点的状态是什么？
* 通过 `nerdctl` 命令更改了哪些容器？
* 通过命令 `kubectl get pod -owide -A` 查看的Pod状态是什么？

演示：

在`cka002`节点上执行命令`systemctl stop kubelet.service`。

在`cka001`或`cka003`上执行命令`kubectl get node`，可以看到`cka002`的状态从`Ready`变为`NotReady`。

在`cka002`上执行命令`nerdctl -n k8s.io container ls`，可以看到所有容器都仍在运行，包括Pod `my-first-pod`。

在`cka002`上执行命令`systemctl start kubelet.service`。

结论：

* 节点状态由`Ready`变为`NotReady`。
* 对于那些类似`calico`、`kube-proxy`这样的 DaemonSet Pod，它们专门在每个节点上运行。在`kubelet`停止后它们不会被终止。
* Pod `my-first-pod` 的状态在每个节点上仍然显示为 `Terminating`，因为状态无法通过`apiserver`从`cka002`同步到其他节点，因为`kubelet`已停止。
* Pod的状态由控制器标记并由`kubelet`回收。
* 当我们在`cka003`上启动`kubelet`服务时，Pod `my-first-pod` 将完全在`cka002`上被终止。

此外，让我们创建一个副本数为3的Deployment。其中两个副本运行在`cka003`上，另一个副本运行在`cka002`上。

```bash
kubectl get pod -o wide -w
```

运行结果

```console
NAME                               READY   STATUS    RESTARTS   AGE    IP           NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-9d745469b-2xdk4   1/1     Running   0          2m8s   10.244.2.3   cka003   <none>           <none>
nginx-deployment-9d745469b-4gvmr   1/1     Running   0          2m8s   10.244.2.4   cka003   <none>           <none>
nginx-deployment-9d745469b-5j927   1/1     Running   0          2m8s   10.244.1.3   cka002   <none>           <none>
```

在我们停止 `cka003` 上的 kubelet 服务后，原先在 `cka003` 上运行的两个副本会被终止，然后会自动在 `cka002` 上创建两个新的副本并运行。

## 监控指标

演示场景：

* 查询pod的监控指标

演示：

查询节点的健康信息。

```bash
kubectl top node
```

运行结果：

```console
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
cka001   147m         7%     1940Mi          50%
cka002   62m          3%     2151Mi          56%
cka003   63m          3%     1825Mi          47%
```

查询pod的监控指标。

```bash
kubectl top pod
```

运行结果：

```console
NAME                                      CPU(cores)   MEMORY(bytes)   
busybox-with-secret                       0m           0Mi
mysql                                     2m           366Mi
mysql-774db46945-sztrp                    2m           349Mi
mysql-nodeselector-6b7d9c875d-227t6       2m           365Mi
mysql-tolerations-5c5986944b-cg9bs        2m           366Mi
mysql-with-sc-pvc-7c97d875f8-dwfkc        2m           349Mi
nfs-client-provisioner-699db7fd58-bccqs   2m           7Mi
nginx                                     0m           3Mi
nginx-app-1-695b7b647d-l76bh              0m           3Mi
nginx-app-2-7f6bf6f4d4-lvbz8              0m           3Mi
nginx-nodename                            0m           3Mi
nginx-with-cm                             0m           3Mi
pod-configmap-env                         0m           3Mi
pod-configmap-env-2                       0m           3Mi
tomcat                                    1m           58Mi
```

通过选项`--sort-by`，对输出结果按照CPU或者内存用量进行排序。

```bash
kubectl top pod --sort-by=cpu
kubectl top pod --sort-by=memory
```

运行结果：

```console
NAME                                      CPU(cores)   MEMORY(bytes)   
nfs-client-provisioner-699db7fd58-bccqs   2m           7Mi
mysql                                     2m           366Mi
mysql-774db46945-sztrp                    2m           349Mi
mysql-nodeselector-6b7d9c875d-227t6       2m           365Mi
mysql-tolerations-5c5986944b-cg9bs        2m           366Mi
mysql-with-sc-pvc-7c97d875f8-dwfkc        2m           349Mi
tomcat                                    1m           58Mi
nginx                                     0m           3Mi
nginx-app-1-695b7b647d-l76bh              0m           3Mi
nginx-app-2-7f6bf6f4d4-lvbz8              0m           3Mi
nginx-nodename                            0m           3Mi
nginx-with-cm                             0m           3Mi
pod-configmap-env                         0m           3Mi
pod-configmap-env-2                       0m           3Mi
busybox-with-secret                       0m           0Mi
```

## 节点驱逐

### 节点的可调度性

演示场景：

* 节点调度

演示：

禁止一个节点的调度。

```bash
kubectl cordon <node_name>
```

举例：

```bash
kubectl cordon cka003
```

输出结果，节点状态如下：

```console
NAME     STATUS                     ROLES                  AGE   VERSION
cka001   Ready                      control-plane,master   18d   v1.24.0
cka002   Ready                      <none>                 18d   v1.24.0
cka003   Ready,SchedulingDisabled   <none>                 18d   v1.24.0
```

激活一个节点的调度。

```bash
kubectl uncordon <node_name>
```

举例：

```bash
kubectl uncordon cka003
```

输出结果，节点状态如下：

```console
NAME     STATUS   ROLES                  AGE   VERSION
cka001   Ready    control-plane,master   18d   v1.24.0
cka002   Ready    <none>                 18d   v1.24.0
cka003   Ready    <none>                 18d   v1.24.0
```

### 驱逐节点

演示内容：

* 驱逐节点 `cka003`

演示：

获取当前运行pod的列表。

```bash
kubectl get pod -o wide
```

其中有一个pod运行在节点`cka003`上。

```console
NAME                                      READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nfs-client-provisioner-86d7fb78b6-xk8nw   1/1     Running   0          22h   10.244.102.3   cka003   <none>           <none>
```

驱逐节点 `cka003`。

```bash
kubectl drain cka003 --ignore-daemonsets --delete-emptydir-data --force
```

输出结果：

```console
node/cka003 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-tr22l, kube-system/kube-proxy-g76kg
evicting pod dev/nfs-client-provisioner-86d7fb78b6-xk8nw
evicting pod cka/cka-demo-64f88f7f46-dkxmk
pod/nfs-client-provisioner-86d7fb78b6-xk8nw evicted
pod/cka-demo-64f88f7f46-dkxmk evicted
node/cka003 drained
```

再次查看pod的状态。

```bash
kubectl get pod -o wide
```

先前运行在节点`cka003`上的pod现在正运行在节点`cka002`上。

```console
NAME                                      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
nfs-client-provisioner-86d7fb78b6-k8xnl   1/1     Running   0          2m20s   10.244.112.4   cka002   <none>           <none>
```

备注：

* `cordon`命令已经包含在`drain`命令中，不需要在执行`drain`之前单独执行`cordon`来禁止node的调度。
