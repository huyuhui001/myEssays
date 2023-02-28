# Job and Cronjob
## Job

!!! Scenario
    * Create Job.

Demo:

Create Job `pi`.
```console
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

Get details of Job.
```console
kubectl get jobs
```

Get details of Job Pod. The status `Completed` means the job was done successfully.
```console
kubectl get pod
```

Get log info of the Job Pod.
```console
kubectl pi-2s74d
3.141592653589793..............
```


Clean up
```console
kubectl delete job pi
```




## Cronjob

!!! Scenario
    * Create Cronjob.

Demo:

Create Cronjob `hello`.
```console
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

Get detail of Cronjob
```console
kubectl get cronjobs -o wide
```
Result
```
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE   CONTAINERS   IMAGES    SELECTOR
hello   */1 * * * *   False     0        <none>          25s   hello        busybox   <none>
```

Monitor Jobs. Every 1 minute a new job will be created. 
```console
kubectl get jobs -w
```

Clean up
```console
kubectl delete cronjob hello
```
