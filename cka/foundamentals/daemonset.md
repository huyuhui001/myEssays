# DaemonSet

!!! Scenario
    * Create DaemonSet.
    * The DaemonSet will run its pod on each node.

Demo:

Create DaemonSet `daemonset-busybox`.
```console
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

Get status of DaemonSet.
```console
kubectl get daemonsets daemonset-busybox
```
Result
```
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-busybox   3         3         3       3            3           <none>          5m33s
```

Get DaemonSet Pod status. It's deployed on each node.
```console
kubectl get pod -o wide
```
Result
```
NAME                      READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
daemonset-busybox-54cj5   1/1     Running   0          44s   10.244.102.4     cka003   <none>           <none>
daemonset-busybox-5tl55   1/1     Running   0          44s   10.244.228.197   cka001   <none>           <none>
daemonset-busybox-wg225   1/1     Running   0          44s   10.244.112.5     cka002   <none>           <none>
```


Clean up.
```console
kubectl delete daemonset daemonset-busybox 
```
