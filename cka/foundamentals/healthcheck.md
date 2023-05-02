# CKA自学笔记26:健康检查

## Pod和Container的状态

演示场景：

* 创建一个有2个容器的pod。

演示：

创建一个包含两个容器 `nginx` 和 `busybox` 的 Pod，命名为 `multi-pods`。

```bash
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

执行下面命令来监控状态，使用选项 `--watch`。
注意，pod的状态已经从`ContainerCreating` 变为 `NotReady`，再变为 `CrashLoopBackOff`。

```bash
kubectl get pod multi-pods --watch
```

获取 Pod `multi-pods` 的详细信息，关注 `Containers` 部分下的容器状态和 `Conditions` 部分下的 Pod 状态。

```bash
kubectl describe pod multi-pods
```

运行结果（部分）：

```yaml
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

演示场景：

* 创建一个pod，内含`livenessProbe`检查。

演示的详细说明可以查询[Kubernetes document](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)。

演示：

创建yaml文件`liveness.yaml`，并包含`livenessProbe`配置，并应用之。

```bash
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

让我们来看看 Pod `liveness-exec` 中发生了什么。

* 创建一个名为 `/tmp/healthy` 的文件夹。
* 执行命令 `cat /tmp/healthy` 并返回成功的代码。
* `35` 秒后，执行命令 `rm -rf /tmp/healthy` 来删除文件夹。探针 `livenessProbe` 检测到失败并返回错误消息。
* kubelet 终止容器并重新启动它。文件夹再次被创建 `touch /tmp/healthy`。

通过命令 `kubectl describe pod liveness-exec`，我们可以看到以下事件消息。
一旦检测到失败，镜像将会再次被拉取，并且文件夹 `/tmp/healthy` 会再次存在。

```console
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

演示场景：

* 创建一个pod，内含 `readinessProbe` 检查。

演示：

Readiness探针的配置与liveness探针类似，唯一的区别是使用`readinessProbe`字段而非`livenessProbe`字段。

创建一个名为`readiness.yaml`的yaml文件并应用其中的`readinessProbe`配置。

```bash
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

pod的Ready状态现在是`0/1`，即，pod并未成功创建。

```bash
kubectl get pod readiness --watch
```

运行结果：

```console
NAME        READY   STATUS    RESTARTS   AGE
readiness   0/1     Running   0          15s
```

执行命令 `kubectl describe pod readiness` 来检查pod的状态，我们可以看到报错信息`Readiness probe failed`。

```console
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

Liveness探针不会等待readiness探针成功后才执行。
如果我们想要在执行Liveness探针之前等待一段时间，可以使用`initialDelaySeconds`或`startupProbe`。

删除演示中创建的临时资源。

```bash
kubectl delete pod liveness-exec
kubectl delete pod multi-pods 
kubectl delete pod readiness
```
