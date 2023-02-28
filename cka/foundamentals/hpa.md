# Horizontal Pod Autoscaling (HPA)

!!! Scenario
    * Install Metrics Server component
    * Create Deployment `podinfo` and Service `podinfo` for stress testing
    * Create HPA `my-hpa`
    * Stress Testing

Demo:

## Install Metrics Server component

Download yaml file for Metrics Server component
```console
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Replace Google image by Aliyun image `image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.1`.
```console
sed -i 's/k8s\.gcr\.io\/metrics-server\/metrics-server\:v0\.6\.1/registry\.aliyuncs\.com\/google_containers\/metrics-server\:v0\.6\.1/g' components.yaml
```

Change `arg` of deployment `metrics-server` by adding `--kubelet-insecure-tls` to disable tls certificate validation. 
```console
vi components.yaml
```
Updated `arg` of `metrics-server` is below.
```
......
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.1
......
```

Appy the yaml file `components.yaml` to deploy `metrics-server`.
```console
kubectl apply -f components.yaml
```
Below resources were crested. 
```
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

Verify if `metrics-server` Pod is running as expected (`1/1` running)
```console
kubectl get pod -n kube-system -owide | grep metrics-server
```
Result
```
NAME                                       READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
metrics-server-7fd564dc66-sdhdc            1/1     Running   0          61s     10.244.102.15    cka003   <none>           <none>
```


Get current usage of CPU, memory of each node.
```console
kubectl top node
```
Result:
```
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
cka001   595m         29%    1937Mi          50%       
cka002   75m          3%     1081Mi          28%       
cka003   79m          3%     1026Mi          26% 
```


## Deploy a Service `podinfo`

Create Deployment `podinfo` and Service `podinfo` for further stress testing.
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  type: NodePort
  ports:
    - port: 9898
      targetPort: 9898
      nodePort: 31198
      protocol: TCP
  selector:
    app: podinfo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfod
        image: stefanprodan/podinfo:0.0.1
        imagePullPolicy: Always
        command:
          - ./podinfo
          - -port=9898
          - -logtostderr=true
          - -v=2
        ports:
        - containerPort: 9898
          protocol: TCP
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "256Mi"
            cpu: "100m"
EOF
```



## Config HPA
 
Create HPA `my-hpa` by setting CPU threshold `50%` to trigger auto-scalling with minimal `2` and maximal `10` Replicas.

Use `kubectl autoscal` to create HPA `my-hpa`.
```
kubectl autoscale deployment podinfo --cpu-percent=50 --min=1 --max=10
```

Use `autoscaling/v1` version template to crreate HPA `my-hpa`. 
```console
kubectl apply -f - <<EOF
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
EOF
```

Use `autoscaling/v2` version template to crreate HPA `my-hpa`, adding memory resource in the matrics.
```console
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 100Mi
EOF
```

Get status of HPA.
```console
kubectl get hpa
```
Result:
```
NAME     REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
my-hpa   Deployment/podinfo   2%/50%    2         10        2          60s
```

!!! Memo
    * `metrics.resource` The values will be averaged together before being compared to the target. 在与目标值比较之前，这些指标值将被平均。
    * `metrics.resource.target.type` represents whether the metric type is Utilization, Value, or AverageValue
    * `metrics.resource.target.averageUtilization` is the target value of the average of the resource metric across all relevant pods, represented as a percentage of the requested value of the resource for the pods. Currently only valid for Resource metric source type. 是跨所有相关 Pod 得出的资源指标均值的目标值。
    * `metrics.resource.target.averageValue` (Quantity) is the target value of the average of the metric across all relevant pods (as a quantity). 跨所有 Pod 得出的指标均值的目标值（以数量形式给出）。
    * `metrics.resource.target.value` (Quantity) is the target value of the metric (as a quantity). 是指标的目标值（以数量形式给出）。

!!! info
    [Algorithm details](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details)



## Stress Testing

### Install ab

Here we will use `ab` tool to simulate 1000 concurrency.

The `ab` command is a command line load testing and benchmarking tool for web servers that allows you to simulate high traffic to a website. 

The short definition form apache.org is: The acronym `ab` stands for Apache Bench where bench is short for benchmarking.

Execute below command to install `ab` tool.
```console
sudo apt install apache2-utils -y
```

Most common options of `ab` are `-n` and `-c`：
```
-n requests     Number of requests to perform
-c concurrency  Number of multiple requests to make at a time
-t timelimit    Seconds to max. to spend on benchmarking. This implies -n 50000
-p postfile     File containing data to POST. Remember also to set -T      
-T content-type Content-type header to use for POST/PUT data, eg. 'application/x-www-form-urlencoded'. Default is 'text/plain'
-k              Use HTTP KeepAlive feature
```

Example: 
```console
ab -n 1000 -c 100 http://www.baidu.com/
```

### Concurrency Stres Test 

Simulate 1000 concurrency request to current node running command `ab`. Node port `31198` is the for the service `podinfo`.
```console
ab -c 1000 -t 60 http://127.0.0.1:31198/
```

By command `kubectl get hpa -w` we can see that CPU workload has been increasing.
```
NAME    REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
......
nginx   Deployment/podinfo   199%/50%    2         10        10         14m
nginx   Deployment/podinfo   934%/50%    2         10        10         14m
nginx   Deployment/podinfo   964%/50%    2         10        10         14m
nginx   Deployment/podinfo   992%/50%    2         10        10         15m
nginx   Deployment/podinfo   728%/50%    2         10        10         15m
nginx   Deployment/podinfo   119%/50%    2         10        10         15m
......
```
And see auto-scalling automically triggered for Deployment `podinfo`.
```console
kubectl get pod
kubectl get deployment
```

Please be noted the scale up is a phased process rather than a sudden event to scale to max. 
And it'll be scaled down to a balanced status when CPU workload is down.
```console
kubectl get hpa -w
```
After several hours, we can see below result with above command.
```
NAME     REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
my-hpa   Deployment/podinfo   2%/50%    2         10        2          60s
```

Clean up.
```console
kubectl delete service podinfo
kubectl delete deployment podinfo
kubectl delete hpa my-hpa
```





