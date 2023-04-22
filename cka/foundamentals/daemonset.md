# CKA自学笔记13:DaemonSet

## 演示场景

* 创建一个DaemonSet
* 创建的DaemonSet会在每个node节点上运行自己的pod。

## 演示

创建 DaemonSet `daemonset-busybox`。

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-busybox
  labels:
    app: daemonset-busybox
spec:
  selector:
    matchLabels:
      app: daemonset-busybox
  template:
    metadata:
      labels:
        app: daemonset-busybox
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: busybox
        image: busybox:1.28
        args:
        - sleep
        - "10000"
EOF

```

获取DaemonSet的运行状态。

```bash
kubectl get daemonsets daemonset-busybox
```

运行结果

```console
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-busybox   3         3         3       3            3           <none>          5m33s
```

获取 DaemonSet 的 Pod 的状态。这些pod会部署在每个节点node上。

```bash
kubectl get pod -o wide
```

运行结果

```bash
NAME                      READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
daemonset-busybox-54cj5   1/1     Running   0          44s   10.244.102.4     cka003   <none>           <none>
daemonset-busybox-5tl55   1/1     Running   0          44s   10.244.228.197   cka001   <none>           <none>
daemonset-busybox-wg225   1/1     Running   0          44s   10.244.112.5     cka002   <none>           <none>
```

删除所创建的资源。

```bash
kubectl delete daemonset daemonset-busybox 
```
