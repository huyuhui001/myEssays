# CKA自学笔记8:Pod

## 摘要

练习目标：

* 创建pod
* 追踪pod
* pod标签
* 静态pod
* 多容器pod
* 含初始化容器的pod

## 创建Pod

创建Pod `my-first-podl`。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
spec:
  containers:
  - name: nginx
    image: nginx:mainline
    ports:
    - containerPort: 80
EOF
```

验证刚刚创建的pod的状态。

```bash
kubectl get pods -o wide
```

## 追踪pod

检查刚刚创建的pod的日志。

```bash
kubectl logs my-first-pod
```

如果日志或者其他命令输出的信息不足以帮助我们查找根本原因，我们可以通过`kubectl exec -it <my-pod> -- bash`来进入pod内部进行分析。

```bash
kubectl exec -it my-first-pod -- bash
root@my-first-pod:/# ls
root@my-first-pod:/# cd bin
root@my-first-pod:/bin# ls
root@my-first-pod:/bin# exit
```

执行命令`kubectl explain pod.spec`可以得到pod对应的yaml文件中Spec区段的内容。

我们可以查看 Pod 资源的官方 API 参考文档，或者使用 `kubectl explain pod` 命令行获取该资源的描述信息。通过在资源类型后添加 `.<field>`，explain 命令会提供该指定字段的更多详细信息。

```bash
kubectl explain pod.kind
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.name
```

## pod的标签

通过选项 `--show-labels`来获得pod的标签。

```bash
kubectl get pods
kubectl get pods --show-labels
```

给pod `pod my-first-pod`添加2个标签。

```bash
kubectl label pod my-first-pod nginx=mainline
kubectl label pod my-first-pod env=demo
kubectl get pods --show-labels
```

通过标签来查找pod。

```bash
kubectl get pod -l env=demo
kubectl get pod -l env=demo,nginx=mainline
kubectl get pod -l env=training
```

移除pod的标签。

```bash
kubectl label pods my-first-pod env-
kubectl get pods --show-labels
```

描述 Pod。

```bash
kubectl describe pod my-first-pod
```

删除pod.
运行命令 `watch kubectl get pods` 来获取pod的状态。

```bash
kubectl delete pod my-first-pod
watch kubectl get pods
```

## 静态pod

演示场景：

* 创建一个静态pod。

* `kubectl` 会自动检查 `/etc/kubernetes/manifests/` 中的 YAML 文件，并在检测到后创建静态 Pod。

演示：

查看系统初始化后已经存在的静态pod。

```bash
ll /etc/kubernetes/manifests/
```

运行结果：

```console
-rw------- 1 root root 2292 Jul 23 10:45 etcd.yaml
-rw------- 1 root root 3889 Jul 23 10:45 kube-apiserver.yaml
-rw------- 1 root root 3395 Jul 23 10:45 kube-controller-manager.yaml
-rw------- 1 root root 1464 Jul 23 10:45 kube-scheduler.yaml
```

在`/etc/kubernetes/manifests/`目录中创建yaml文件`my-nginx.yaml`，一旦文件创建完成，静态Pod `my-nginx` 将会被自动创建。

```bash
kubectl run my-nginx --image=nginx:mainline --dry-run=client -n default -oyaml | sudo tee /etc/kubernetes/manifests/my-nginx.yaml
```

检查 Pod `my-nginx` 的状态。Pod 名称中包含节点名称 `cka001`，这意味着该 Pod 正在节点 `cka001` 上运行。

```bash
kubectl get pod -o wide
```

运行结果：

```console
NAME              READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
my-nginx-cka001   1/1     Running   0          20s   10.244.228.196   cka001   <none>           <none>
```

删除 `/etc/kubernetes/manifests/my-nginx.yaml` 这个 yaml 文件，对应的静态 Pod `my-nginx` 将会被自动删除。

```bash
sudo rm /etc/kubernetes/manifests/my-nginx.yaml 
```

## 多容器Pod

演示场景：

* 创建多容器Pod
* 描述该Pod
* 检查Pod的日志
* 检查容器的日志

演示：

创建一个名为`multi-container-pod`的Pod，包含多个容器：`container-1-nginx`和`container-2-alpine`。

```yaml
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: container-1-nginx
    image: nginx
    ports:
    - containerPort: 80  
  - name: container-2-alpine
    image: alpine
    command: ["watch", "wget", "-qO-", "localhost"]
EOF
```

获取pod状态。

```bash
kubectl get pod multi-container-pod
```

运行结果

```console

```

获取pod的详细信息。

```bash
kubectl describe pod multi-container-pod
```

运行结果：

```console
.......
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  41s   default-scheduler  Successfully assigned dev/multi-container-pod to cka002
  Normal  Pulling    40s   kubelet            Pulling image "nginx"
  Normal  Pulled     23s   kubelet            Successfully pulled image "nginx" in 16.767129903s
  Normal  Created    22s   kubelet            Created container container-1-nginx
  Normal  Started    22s   kubelet            Started container container-1-nginx
  Normal  Pulling    22s   kubelet            Pulling image "alpine"
  Normal  Pulled     14s   kubelet            Successfully pulled image "alpine" in 7.776104353s
  Normal  Created    14s   kubelet            Created container container-2-alpine
  Normal  Started    14s   kubelet            Started container container-2-alpine
```

对于多容器 Pod，如果我们想通过命令 `kubectl logs <pod_name> <container_name>` 获取 Pod 的日志，需要指定容器名称。如果不指定容器名称，将会收到错误信息。

```bash
kubectl logs multi-container-pod
```

运行结果

```bash
error: a container name must be specified for pod multi-container-pod, choose one of: [container-1-nginx container-2-alpine]
```

指定容器名称，我们可以得到对应的日志信息。

```bash
kubectl logs multi-container-pod container-1-nginx
```

运行结果

```bash
......
::1 - - [23/Jul/2022:04:06:37 +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
```

如果我们需要使用命令 `kubectl exec -it <pod_name> -c <container_name> -- <commands>` 登录到 Pod 中，同样需要指定容器名称。如果没有指定容器名称，会出现错误。

```bash
kubectl exec -it multi-container-pod -c container-1-nginx -- /bin/bash
root@multi-container-pod:/# ls
```

删除上面练习中创建的pod。

```bash
kubectl delete pod multi-container-pod
```

下面是一个基本的yaml文件用来创建多容器pod。

```yaml
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-multi-pod
spec:
  containers:
  - image: nginx
    name: nginx
  - image: memcached
    name: memcached
  - image: redis
    name: redis
EOF
```

演示场景：

* 创建一个名为`my-busybox`的Pod，并在其中添加一个名为`container-1-busybox`的容器。该容器将把消息写入到文件`/var/log/my-pod-busybox.log`中。
* 向Pod `my-busybox`中添加另一个容器`container-2-busybox`（Sidecar）。Sidecar容器从文件`/var/log/my-pod-busybox.log`中读取消息。
* 提示：创建一个Volume来存储日志文件，并与两个容器共享。

演示：

创建一个名为 `my-busybox` 的 Pod，其中包含一个容器 `container-1-busybox`。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-busybox
spec:
  containers:
  - name: container-1-busybox
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "Hello message from container-1: $i" >> /var/log/my-pod-busybox.log;
        i=$((i+1));
        sleep 1;
      done
EOF
```

在 Kubernetes 文档中搜索 `emptyDir`。
参考以下模板用于 `emptyDir`：<https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/>

将以下新功能添加到 Pod 中：

* Volume：
  * 卷名称：`logfile`
  * 类型：`emptyDir`
* 更新现有容器：
  * name: `container-1-busybox`
  * volumeMounts
    * name: `logfile`
    * mounthPath: `/var/log`
* 添加新容器：
  * name: `container-2-busybox`
  * image: busybox
  * args: ['/bin/sh', '-c', 'tail -n+1 -f /var/log/my-pod-busybox.log']
  * volumeMounts:
    * name: `logfile`
    * mountPath: `/var/log`

```bash
kubectl get pod my-busybox -o yaml > my-busybox.yaml
vi my-busybox.yaml
kubectl delete pod my-busybox 
kubectl apply -f my-busybox.yaml
kubectl logs my-busybox -c container-2-busybox
```

更新后的文件`my-busybox.yaml`如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/containerID: 89644b6b073cd152f94b4cae7bdea6bbc3292cf59afd4f28102bd74f0205c9e4
    cni.projectcalico.org/podIP: 10.244.102.20/32
    cni.projectcalico.org/podIPs: 10.244.102.20/32
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"my-busybox","namespace":"dev"},"spec":{"containers":[{"args":["/bin/sh","-c","i=0; while true; do\n  echo \"Hello message from container-1: \" \u003e\u003e /var/log/my-pod-busybox.log;\n  i=1;\n  sleep 1;\ndone\n"],"image":"busybox","name":"container-1-busybox"}]}}
  creationTimestamp: "2022-07-29T22:58:27Z"
  name: my-busybox
  namespace: dev
  resourceVersion: "1185720"
  uid: c5e62a16-4459-4828-a441-7d1471b89a56
spec:
  containers:
  - name: container-2-busybox
    image: busybox
    args: ['/bin/sh', '-c', 'tail -n+1 -f /var/log/my-pod-busybox.log']
    volumeMounts:
    - name: logfile
      mountPath: /var/log
  - args:
    - /bin/sh
    - -c
    - |
      i=0; while true; do
        echo "Hello message from container-1: $i" >> /var/log/my-pod-busybox.log;
        i=1;
        sleep 1;
      done
    image: busybox
    imagePullPolicy: Always
    name: container-1-busybox
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - name: logfile
      mountPath: /var/log
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-mhxlf
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: cka003
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: logfile
    emptyDir: {}
  - name: kube-api-access-mhxlf
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-07-29T22:58:27Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2022-07-29T22:58:30Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-07-29T22:58:30Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2022-07-29T22:58:27Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://fd42d4ba4d94d8918d8846327b1db2328be13c5f93f381877ff0228ed7b5468d
    image: docker.io/library/busybox:latest
    imageID: docker.io/library/busybox@sha256:0e97a8ca6955f22dbc7db9e9dbe970971f423541e52c34b8cb96ccc88d6a3883
    lastState: {}
    name: container-1-busybox
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-07-29T22:58:30Z"
  hostIP: <cka003_ip>
  phase: Running
  podIP: 10.244.102.20
  podIPs:
  - ip: 10.244.102.20
  qosClass: BestEffort
  startTime: "2022-07-29T22:58:27Z"
```

清理上面练习中创建的pod。

```bash
kubectl delete pod my-busybox
```

## 含初始化容器Pod

演示场景：

* 创建拥有两个初始化容器的 Pod `myapp-pod`。
  * `myapp-container`
  * `init-mydb`
* 创建两个服务：
  * `myservice`
  * `mydb`

演示预期结论：

* `myapp-container`等待服务`myservice`启动，以解析名称`myservice.dev.svc.cluster.local`
* `init-mydb`等待服务`mydb`启动，以解析名称`mydb.dev.svc.cluster.local`。

演示：

创建名为`myapp-pod.yaml`的yaml文件，并添加以下内容。
注意：由于命令`$(cat /var/.....`将被视为主机变量，因此我们不能使用echo生成该文件。它是容器本身的变量。

```bash
vi myapp-pod.yaml
```

文件内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

用上面创建的yaml文件创建Pod `myapp-pod`。

```bash
kubectl apply -f myapp-pod.yaml
```

检查pod的状态。

```bash
kubectl get pod myapp-pod
```

运行结果：

```console
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:0/2   0          12m
```

检查Pod，可以看到两个错误：

* nslookup: 无法解析'myservice.dev.svc.cluster.local'
* Pod "myapp-pod"中的容器“init-mydb”正在等待启动：PodInitializing

```bash
kubectl logs myapp-pod -c init-myservice # Inspect the first init container
kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

在这个时候，这些 init 容器将等待发现名为 `mydb` 和 `myservice` 的服务。

创建 `mydb` 和 `myservice` 服务：

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
EOF
```

查看创建的服务的状态。

```bash
kubectl get service
```

创建的2个服务都是运行状态。

```console
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
mydb        ClusterIP   11.244.239.149   <none>        80/TCP    6s
myservice   ClusterIP   11.244.116.126   <none>        80/TCP    6s
```

再次查看pod的状态。

```bash
kubectl get pod myapp-pod -o wide
```

pod已经正常运行了。

```console
NAME        READY   STATUS     RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
myapp-pod   0/1     Init:0/2   0          2m40s   10.244.112.2   cka002   <none>           <none>
```

现在我们可以看到那些初始化容器都已经完成，`myapp-pod` Pod 进入了 Running 状态。

删除上面练习中创建的pod。

```bash
kubectl delete service mydb myservice 
kubectl delete pod myapp-pod 
```

参考：

* [Pod basics](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
* [Lifecycle & phases](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [Kubernetes pod design pattern](https://www.cnblogs.com/zhenyuyaodidiao/p/6514907.html)
