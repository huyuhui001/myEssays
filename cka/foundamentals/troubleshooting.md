# Troubleshooting

## Event

!!! Scenario
    * Describe pod to get event information.

Demo:

Usage:
```console
kubectl describe <resource_type> <resource_name> --namespace=<namespace_name>
```

Get event information of a Pod

Create a Tomcat Pod.
```console
kubectl run tomcat --image=tomcat
```

Check event of above deplyment.
```console
kubectl describe pod/tomcat
```
Get below event information.
```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  55s   default-scheduler  Successfully assigned dev/tomcat to cka002
  Normal  Pulling    54s   kubelet            Pulling image "tomcat"
  Normal  Pulled     21s   kubelet            Successfully pulled image "tomcat" in 33.134162692s
  Normal  Created    19s   kubelet            Created container tomcat
  Normal  Started    19s   kubelet            Started container tomcat
```

Get event information for a Namespace.
```console
kubectl get events -n <your_namespace_name>
```
Get current default namespace event information.
```
LAST SEEN   TYPE      REASON           OBJECT                          MESSAGE
70s         Warning   FailedGetScale   horizontalpodautoscaler/nginx   deployments/scale.apps "podinfo" not found
2m16s       Normal    Scheduled        pod/tomcat                      Successfully assigned dev/tomcat to cka002
2m15s       Normal    Pulling          pod/tomcat                      Pulling image "tomcat"
102s        Normal    Pulled           pod/tomcat                      Successfully pulled image "tomcat" in 33.134162692s
100s        Normal    Created          pod/tomcat                      Created container tomcat
100s        Normal    Started          pod/tomcat                      Started container tomcat
```

Get event information for all Namespace.
```console
kubectl get events -A
```




## Logs

!!! Scenario
    * Get log of pod


Usage:
```console
kubectl logs <pod_name> -n <namespace_name>
```

Options:

* `--tail <n>`: display only the most recent `<n>` lines of output
* `-f`: streaming the output

Get the most recent 100 lines of output.
```console
kubectl logs -f tomcat --tail 100
```

If it's multipPod, use `-c` to specify Container.
```console
kubectl logs -f tomcat --tail 100 -c tomcat
```


## Node Availability


### Check Available Node

!!! Scenario
    * Check node availibility.

Demo: 

Option 1:
```console
kubectl describe node | grep -i taint
```
Manual check the result, here it's `2` nodes are available
```
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             <none>
Taints:             <none>
```

Option 2:
```console
kubectl describe node | grep -i taint |grep -vc NoSchedule
```
We will get same result `2`. Here `-v` means exclude, `-c` count numbers.



### Node NotReady

!!! Scenario 
    When we stop `kubelet` service on worker node `cka002`,
    * What's the status of each node?
    * What's containers changed via command `nerdctl`?
    * What's pods status via command `kubectl get pod -owide -A`? 

Demo:

Execute command `systemctl stop kubelet.service` on `cka002`.

Execute command `kubectl get node` on either `cka001` or `cka003`, the status of `cka002` is `NotReady`.

Execute command `nerdctl -n k8s.io container ls` on `cka002` and we can observe all containers are still up and running, including the pod `my-first-pod`.

Execute command `systemctl start kubelet.service` on `cka002`.


!!! Conclusion
    * The node status is changed to `NotReady` from `Ready`.
    * For those DaemonSet pods, like `calico`、`kube-proxy`, are exclusively running on each node. They won't be terminated after `kubelet` is down.
    * The status of pod `my-first-pod` keeps showing `Terminating` on each node because status can not be synced to other nodes via `apiserver` from `cka002` because `kubelet` is down.
    * The status of pod is marked by `controller` and recycled by `kubelet`.
    * When we start kubelet service on `cka003`, the pod `my-first-pod` will be termiated completely on `cka002`.

In addition, let's create a deployment with 3 replicas. Two are running on `cka003` and one is running on `cka002`.
```console
root@cka001:~# kubectl get pod -o wide -w
NAME                               READY   STATUS    RESTARTS   AGE    IP           NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-9d745469b-2xdk4   1/1     Running   0          2m8s   10.244.2.3   cka003   <none>           <none>
nginx-deployment-9d745469b-4gvmr   1/1     Running   0          2m8s   10.244.2.4   cka003   <none>           <none>
nginx-deployment-9d745469b-5j927   1/1     Running   0          2m8s   10.244.1.3   cka002   <none>           <none>
```
After we stop kubelet service on `cka003`, the two running on `cka003` are terminated and another two are created and running on `cka002` automatically. 




## Monitoring Indicators

!!! Scenario
    * Get monitoring indicators of pod


Demo:

Get node monitoring information
```console
kubectl top node
```
Output:
```
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
cka001   147m         7%     1940Mi          50%
cka002   62m          3%     2151Mi          56%
cka003   63m          3%     1825Mi          47%
```

Get Pod monitoring information
```console
kubectl top pod
```
Output:
```
root@cka001:~# kubectl top pod
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

Sort output by CPU or Memory using option `--sort-by`, the field can be either 'cpu' or 'memory'.
```console
kubectl top pod --sort-by=cpu
kubectl top pod --sort-by=memory
```
Output:
```
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







## Node Eviction

### Cordon/Uncordon

!!! Scenario
    * Scheduling for a node

Demo:

Disable scheduling for a Node.
```console
kubectl cordon <node_name>
```
Example:
```console
kubectl cordon cka003
```
Node status:
```
NAME     STATUS                     ROLES                  AGE   VERSION
cka001   Ready                      control-plane,master   18d   v1.24.0
cka002   Ready                      <none>                 18d   v1.24.0
cka003   Ready,SchedulingDisabled   <none>                 18d   v1.24.0
```

Enable scheduling for a Node.
```console
kubectl uncordon <node_name>
```
Example:
```console
kubectl uncordon cka003
```
Node status:
```
NAME     STATUS   ROLES                  AGE   VERSION
cka001   Ready    control-plane,master   18d   v1.24.0
cka002   Ready    <none>                 18d   v1.24.0
cka003   Ready    <none>                 18d   v1.24.0
```



### Drain Node

!!! Scenario
    * Drain the node `cka003`

Demo:

Get list of Pods running.
```console
kubectl get pod -o wide
```

We know that a Pod is running on `cka003`.
```
NAME                                      READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nfs-client-provisioner-86d7fb78b6-xk8nw   1/1     Running   0          22h   10.244.102.3   cka003   <none>           <none>
```

Evict node `cka003`.
```console
kubectl drain cka003 --ignore-daemonsets --delete-emptydir-data --force
```
Output looks like below.
```
node/cka003 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-tr22l, kube-system/kube-proxy-g76kg
evicting pod dev/nfs-client-provisioner-86d7fb78b6-xk8nw
evicting pod cka/cka-demo-64f88f7f46-dkxmk
pod/nfs-client-provisioner-86d7fb78b6-xk8nw evicted
pod/cka-demo-64f88f7f46-dkxmk evicted
node/cka003 drained
```

Check pod status again.
```console
kubectl get pod -o wide
```
The pod is running on `cka002` now.
```
NAME                                      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
nfs-client-provisioner-86d7fb78b6-k8xnl   1/1     Running   0          2m20s   10.244.112.4   cka002   <none>           <none>
```


!!! Note
    * `cordon` is included in `drain`, no need additional step to `cordon` node before `drain` node. 



