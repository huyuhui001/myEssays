# CKA自学笔记14:Job and Cronjob

## Job

演示场景：

* 创建Job。

演示：

创建Job `pi`。

```bash
kubectl apply -f - << EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
EOF

```

获取Job的详细信息。

```bash
kubectl get jobs
```

获取Job的Pod的详细信息。 `Completed` 的状态代表这个job已经成功完成了。

```bash
kubectl get pod
```

获取Job的Pod的日志信息。

```bash
kubectl pi-2s74d
3.141592653589793..............
```

删除所创建的资源。

```bash
kubectl delete job pi
```

## Cronjob

演示场景：

* 创建Cronjob。

演示：

创建Cronjob `hello`。

```bash
kubectl apply -f - << EOF
apiVersion: batch/v1
kind: CronJob
metadata:
 name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
   spec:
    template:
     spec:
      containers:
      - name: hello
        image: busybox
        args:
        - /bin/sh
        - -c
        - date ; echo Hello from the kubernetes cluster
      restartPolicy: OnFailure
EOF

```

获取Cronjob的详细信息。

```bash
kubectl get cronjobs -o wide
```

运行结果

```console
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE   CONTAINERS   IMAGES    SELECTOR
hello   */1 * * * *   False     0        <none>          25s   hello        busybox   <none>
```

监控Jobs。每隔1分钟，一个新的job会被创建。

```bash
kubectl get jobs -w
```

删除创建的资源。

```bash
kubectl delete cronjob hello
```
