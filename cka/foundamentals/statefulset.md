# StatefulSet

!!! Scenario
    * Create Headless Service `nginx` and StatefulSet `web`
    * Scale out StatefulSet `web`

Demo:

Create Headless Service `nginx` and StatefulSet `web`.
```console
kubectl apply -f - << EOF
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
EOF
```

Get details of StatefulSet Pod created just now.
```console
kubectl get pod | grep web
```
Result
```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          27s
web-1   1/1     Running   0          10s
```

Use command `kubectl edit sts web` to update an existing StatefulSet.
ONLY these fields can be updated: `replicas`、`image`、`rolling updates`、`labels`、`resource request/limit` and `annotations`.

Note: 
When StatefulSet Pod is dead in current node, no copies will be created in other node automatically.


Scale out StatefulSet. 

Scale StatefulSet `web` to `5` Replicas.
```console
kubectl scale sts web --replicas=5
```

!!! Info
    Partition indicates the ordinal at which the StatefulSet should be partitioned for updates. During a rolling update, all pods from ordinal Replicas-1 to Partition are updated. All pods from ordinal Partition-1 to 0 remain untouched. 
    This is helpful in being able to do a canary based deployment. 
    The default value is 0.
    
    Command: `kubectl explain statefulsets.spec.updateStrategy.rollingUpdate.partition`



Clean up.
```console
kubectl delete sts web
kubectl delete service nginx
```

