# Work on pod

## Create pod

Create pod `my-first-podl`.
```console
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

Verify status of the pod just created.
```console
kubectl get pods -o wide
```


## Track pod

Check logs of the pod just created.
```console
kubectl logs my-first-pod
```

In case logs or describe or any other of the output generating commands don't help us to get to the root cause of an issue, we can use use `kubectl exec -it <my-pod> -- bash` command to look into it ourselves.
```console
kubectl exec -it my-first-pod -- bash
root@my-first-pod:/# ls
root@my-first-pod:/# cd bin
root@my-first-pod:/bin# ls
root@my-first-pod:/bin# exit
```


Execute command `kubectl explain pod.spec` will get details of Spec segment of Pod kind in yaml file.

We can check the official API reference of the pod resource for help or use `kubectl explain pod` to get a command-line based description of the resource. 
By appending .<field> to the resource type, the explain command will provide more details on the specified field.
```console
kubectl explain pod.kind
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.name
```




## Label pod

Get pod's label with option `--show-labels`.
```console
kubectl get pods
kubectl get pods --show-labels
```

Add two labels to the pod `pod my-first-pod`.
```console
kubectl label pod my-first-pod nginx=mainline
kubectl label pod my-first-pod env=demo
kubectl get pods --show-labels
```

Search pod by labels.
```console
kubectl get pod -l env=demo
kubectl get pod -l env=demo,nginx=mainline
kubectl get pod -l env=training
```

Remove label
```console
kubectl label pods my-first-pod env-
kubectl get pods --show-labels
```

Describe pod.
```console
kubectl describe pod my-first-pod
```

Delete pod.
Run `watch kubectl get pods` to monitor the pod status. 
```console
kubectl delete pod my-first-pod
watch kubectl get pods
```

## Static Pod

!!! scenario
    * Create a static pod.
    * `kubectl` will automatically check yaml file in `/etc/kubernetes/manifests/` and create the static pod once it's detected.

Demo: 

Some system static Pods are already in place.
```console
ll /etc/kubernetes/manifests/
```
Result
```
-rw------- 1 root root 2292 Jul 23 10:45 etcd.yaml
-rw------- 1 root root 3889 Jul 23 10:45 kube-apiserver.yaml
-rw------- 1 root root 3395 Jul 23 10:45 kube-controller-manager.yaml
-rw------- 1 root root 1464 Jul 23 10:45 kube-scheduler.yaml
```

Create yaml file `my-nginx.yaml` in directory `/etc/kubernetes/manifests/`. Once the file is created, the static pod `my-nginx` is created automatically.
```console
kubectl run my-nginx --image=nginx:mainline --dry-run=client -n default -oyaml | sudo tee /etc/kubernetes/manifests/my-nginx.yaml
```

Check status of the Pod `my-nginx`. The node name `cka001` is part of the Pod name, which means the Pod is running on node `cka001`.
```console
kubectl get pod -o wide
```
Result
```
NAME              READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
my-nginx-cka001   1/1     Running   0          20s   10.244.228.196   cka001   <none>           <none>
```

Delete the yaml file `/etc/kubernetes/manifests/my-nginx.yaml`, the static pod will be deleted automatically.
```console
sudo rm /etc/kubernetes/manifests/my-nginx.yaml 
```





## Multi-Container Pod

!!! Scenario
    * Create Multi-Container Pod
    * Describe the Pod
    * Check the log of Pod
    * Check the log of Containers


Create a Pod `multi-container-pod` with multiple container `container-1-nginx` and `container-2-alpine`.
```console
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

Get the status.
```console
kubectl get pod multi-container-pod
```
Result
```
NAME                  READY   STATUS    RESTARTS   AGE
multi-container-pod   2/2     Running   0          81s
```

Get details of the pod.
```console
kubectl describe pod multi-container-pod
```
Result
```
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

For multi-container pod, container name is needed if we want to get log of pod via command `kubectl logs <pod_name> <container_name>`.

Without the container name, we receive error.
```console
kubectl logs multi-container-pod
```
Result
```
error: a container name must be specified for pod multi-container-pod, choose one of: [container-1-nginx container-2-alpine]
```

With specified container name, we get the log info.
```console
kubectl logs multi-container-pod container-1-nginx
```
Result
```
......
::1 - - [23/Jul/2022:04:06:37 +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
```

Same if we need specify container name to login into the pod via command `kubectl exec -it <pod_name> -c <container_name> -- <commands>`.
```console
kubectl exec -it multi-container-pod -c container-1-nginx -- /bin/bash
root@multi-container-pod:/# ls
```

Clean up
```console
kubectl delete pod multi-container-pod
```

Quick reference of a simple yaml file to create Multiple Containers.
```console
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



!!! Scenario
    * Create a Pod `my-busybox` with a container `container-1-busybox`. The container will write message to file `/var/log/my-pod-busybox.log`.
    * Add another container `container-2-busybox` (Sidecar) to the Pod `my-busybox`. The sidecar container read message from file `/var/log/my-pod-busybox.log`.
    * Tips: create a Volume to store the log file, which is shared with two containers.

Demo:

Create a Pod `my-busybox` with a container `container-1-busybox`.
```console
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

Search `emptyDir` in the Kubernetes documetation.
Refer to below template for emptyDir via https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/

Add below new features into the Pod

* Volume:
    * volume name: `logfile`
    * type: `emptyDir`
* Update existing container:
    * name: `container-1-busybox`
    * volumeMounts
        * name: `logfile`
        * mounthPath: `/var/log`
* Add new container:
    * name: `container-2-busybox`
    * image: busybox
    * args: ['/bin/sh', '-c', 'tail -n+1 -f /var/log/my-pod-busybox.log']
    * volumeMounts:
        * name: `logfile`
        * mountPath: `/var/log`

```console
kubectl get pod my-busybox -o yaml > my-busybox.yaml
vi my-busybox.yaml
kubectl delete pod my-busybox 
kubectl apply -f my-busybox.yaml
kubectl logs my-busybox -c container-2-busybox
```


The final file `my-busybox.yaml` looks like below.
```console
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


Clean up:
```console
kubectl delete pod my-busybox
```





## initContainer Pod

!!! Scenario
    * Create Pod `myapp-pod` that has two init containers. 
        * `myapp-container`
        * `init-mydb`
    * Create two Services.
        * `myservice`
        * `mydb`

!!! Conclusion
    * `myapp-container` waits for Service `myservice` up in order to resolve the name `myservice.dev.svc.cluster.local`
    * `init-mydb` waits for Service `mydb` up in order to resolve the name `mydb.dev.svc.cluster.local`.

Demo: 

Create Pod `myapp-pod` with below yaml file.

Create yaml file `myapp-pod.yaml` and add below content. 
Note: Due to the command `$(cat /var/.....` will be treated as host variable, we can not use echo to generate the file. It's the variabel in container itself.
```console
vi myapp-pod.yaml
```
Content
```console
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

Apply the yaml file to create the Pod.
```console
kubectl apply -f myapp-pod.yaml
```

Check Pod status.
```console
kubectl get pod myapp-pod
```
Result
```
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:0/2   0          12m
```

Inspect Pods. Get two errors:

* nslookup: can't resolve 'myservice.dev.svc.cluster.local'
* container "init-mydb" in pod "myapp-pod" is waiting to start: PodInitializing

```console
kubectl logs myapp-pod -c init-myservice # Inspect the first init container
kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

At this point, those init containers will be waiting to discover Services named mydb and myservice.

Create the `mydb` and `myservice` services:
```console
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

Get current status of Services.
```console
kubectl get service
```
Both of Services are up.
```
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
mydb        ClusterIP   11.244.239.149   <none>        80/TCP    6s
myservice   ClusterIP   11.244.116.126   <none>        80/TCP    6s
```

Get current Pod status.
```console
kubectl get pod myapp-pod -o wide
```
The Pod is up.
```
NAME        READY   STATUS     RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
myapp-pod   0/1     Init:0/2   0          2m40s   10.244.112.2   cka002   <none>           <none>
```

We now see that those init containers complete, and that the myapp-pod Pod moves into the Running state.

Clean up.
```console
kubectl delete service mydb myservice 
kubectl delete pod myapp-pod 
```



!!! References
    * [Pod basics](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
    * [Lifecycle & phases](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
    * [Kubernetes pod design pattern](https://www.cnblogs.com/zhenyuyaodidiao/p/6514907.html)
