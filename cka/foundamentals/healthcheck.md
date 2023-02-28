# Health Check

## Status of Pod and Container

!!! Scenario
    Create a pod with two containers.

Demo:

Create a Pod `multi-pods` with two containers `nginx` and `busybox`. 
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi-pods
  name: multi-pods
spec:
  containers:
  - image: nginx
    name: nginx
  - image: busybox
    name: busybox
  dnsPolicy: ClusterFirst
  restartPolicy: Always
EOF
```

Minotor the status with option `--watch`. The status of Pod was changed from `ContainerCreating` to `NotReady` to `CrashLoopBackOff`.
```console
kubectl get pod multi-pods --watch
```

Get details of the Pod `multi-pods`, focus on Container's state under segment `Containers` and Conditions of Pod under segment `Conditions`.
```console
kubectl describe pod multi-pods
```
Result
```
......
Containers:
  nginx:
    ......
    State:          Running
      Started:      Sat, 23 Jul 2022 15:06:56 +0800
    Ready:          True
    Restart Count:  0
    ......
  busybox:
    ......
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
......
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
...... 
```





## LivenessProbe

!!! Scenario
    Create pod with `livenessProbe` check.

Detail description of the demo can be found on the [Kubernetes document](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

Demo:

Create a yaml file `liveness.yaml` with `livenessProbe` setting and apply it.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

```

Let's see what happened in the Pod `liveness-exec`.

* Create a folder `/tmp/healthy`.
* Execute the the command `cat /tmp/healthy` and return successful code.
* After `35` seconds, execute command `rm -rf /tmp/healthy` to delete the folder. The probe `livenessProbe` detects the failure and return error message.
* The kubelet kills the container and restarts it. The folder is created again `touch /tmp/healthy`.

By command `kubectl describe pod liveness-exec`, wec can see below event message. 
Once failure detected, image will be pulled again and the folder `/tmp/healthy` is in place again.
```
......
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  63s                default-scheduler  Successfully assigned dev/liveness-exec to cka002
  Normal   Pulling    62s                kubelet            Pulling image "busybox"
  Normal   Pulled     53s                kubelet            Successfully pulled image "busybox" in 9.153404392s
  Normal   Created    53s                kubelet            Created container liveness
  Normal   Started    53s                kubelet            Started container liveness
  Warning  Unhealthy  12s (x3 over 22s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    12s                kubelet            Container liveness failed liveness probe, will be restarted
```




## ReadinessProbe

!!! Scenario
    Create a pod with `readinessProbe` check.

Demo: 

Readiness probes are configured similarly to liveness probes. 
The only difference is that you use the readinessProbe field instead of the livenessProbe field.

Create a yaml file `readiness.yaml` with `readinessProbe` setting and apply it.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: readiness
spec:
    containers:
    - name: readiness
      image: busybox
      args:
      - /bin/sh
      - -c
      - touch /tmp/healthy; sleep 5;rm -rf /tmp/healthy; sleep 600
      readinessProbe:
        exec:
          command:
          - cat
          - /tmp/healthy
        initialDelaySeconds: 10
        periodSeconds: 5
EOF

```

The ready status of the Pod is 0/1, that is, the Pod is not up successfully.
```console
kubectl get pod readiness --watch
```
Result
```
NAME        READY   STATUS    RESTARTS   AGE
readiness   0/1     Running   0          15s
```

Execute command `kubectl describe pod readiness` to check status of Pod. 
We see failure message `Readiness probe failed`.
```
......
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  46s               default-scheduler  Successfully assigned dev/readiness to cka002
  Normal   Pulling    45s               kubelet            Pulling image "busybox"
  Normal   Pulled     43s               kubelet            Successfully pulled image "busybox" in 2.244369424s
  Normal   Created    43s               kubelet            Created container readiness
  Normal   Started    43s               kubelet            Started container readiness
  Warning  Unhealthy  1s (x7 over 31s)  kubelet            Readiness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```


Liveness probes do not wait for readiness probes to succeed. 
If we want to wait before executing a liveness probe you should use initialDelaySeconds or a startupProbe.


Clean up.
```console
kubectl delete pod liveness-exec
kubectl delete pod multi-pods 
kubectl delete pod readiness
```






