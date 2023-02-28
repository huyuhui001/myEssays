# Policy

## ResourceQuota

!!! Scenario
    * Create ResourceQuota `object-quota-demo` for namespace `quota-object-example`.
    * Test ResourceQuota `object-quota-demo` for NodePort
    * Test ResourceQuota `object-quota-demo` for PVC


### Create Namespace

Ceate a Namespace
```console
kubectl create ns quota-object-example
```

### Create ResourceQuota for Namespace

Create ResourceQuota `object-quota-demo` for namespace `quota-object-example`.
Within the namespace, we can only create 1 PVC, 1 LoadBalancer Service, can not create NodePort Service.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota-demo
  namespace: quota-object-example
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
EOF
```


### Check Quota status

```console
kubectl get resourcequota object-quota-demo --namespace=quota-object-example --output=yaml
```
Key information is below. 
```
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
status:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
  used:
    persistentvolumeclaims: "0"
    services.loadbalancers: "0"
    services.nodeports: "0"
```

### Test Quota for NodePort

Create a Deployment `ns-quota-test` on namespace `quota-object-example`.
```console
kubectl create deployment ns-quota-test --image nginx --namespace=quota-object-example
```

Expose the Deployment via NodePort    
```console
kubectl expose deployment ns-quota-test --port=80 --type=NodePort --namespace=quota-object-example
```

We receive below error, which is expected because we set Quota `services.nodeports: 0`.
```
Error from server (Forbidden): services "ns-quota-test" is forbidden: exceeded quota: object-quota-demo, requested: services.nodeports=1, used: services.nodeports=0, limited: services.nodeports=0
```
  

### Test Quota for PVC

Create a PVC `pvc-quota-demo` on namespace `quota-object-example`.
```console
kubectl applly -f - << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-quota-demo
  namespace: quota-object-example
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
EOF
```

Check the Quota status.
```console
kubectl get resourcequota object-quota-demo --namespace=quota-object-example --output=yaml
```
Here `persistentvolumeclaims` is used `1`, and the quota is also `1`. If we create PVC again, will receive 403 error.
```
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
status:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
  used:
    persistentvolumeclaims: "1"
    services.loadbalancers: "0"
    services.nodeports: "0"
```




## LimitRange

!!! Scenario
    * Create LimitRange `cpu-limit-range` to define range of CPU Request and CPU Limit for a Container. 
    * Test LimitRange `cpu-limit-range` via Pod.
        * Scenario 1: Pod without specified limits
        * Scenario 2: Pod with CPU limit, without CPU Request
        * Scenario 3: Pod with CPU Request onlyl, without CPU Limits


!!! Background
    A *LimitRange* provides constraints that can:
    
    * Enforce minimum and maximum compute resources usage per Pod or Container in a namespace.
    * Enforce minimum and maximum storage request per PersistentVolumeClaim in a namespace.
    * Enforce a ratio between request and limit for a resource in a namespace.
    * Set default request/limit for compute resources in a namespace and automatically inject them to Containers at runtime.


### Set LimitRange

Create a Namespace `default-cpu-example` for demo.
```console
kubectl create namespace default-cpu-example
```

Create LimitRange `cpu-limit-range` to define range of CPU Request and CPU Limit for a Container.
After apply LimitRange resource, the CPU limitation will affect all new created Pods.
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
  namespace: default-cpu-example
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
EOF
```



### Test via Pod

* Scenario 1: Pod without specified limits

Create a Pod without any specified limits.
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo
  namespace: default-cpu-example
spec:
  containers:
  - name: default-cpu-demo-ctr
    image: nginx
EOF
```

Verify details of the Pod we created. The Pod inherits the both CPU Limits and CPU Requests from namespace as its default.
```console
kubectl get pod default-cpu-demo --output=yaml --namespace=default-cpu-example
```
```
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: default-cpu-demo-ctr
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: 500m
```



* Scenario 2: Pod with CPU limit, without CPU Request

Create Pod with specified CPU limits only.  
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-limit
  namespace: default-cpu-example
spec:
  containers:
  - name: default-cpu-demo-limit-ctr
    image: nginx
    resources:
      limits:
        cpu: "1"
EOF

kubectl apply -f default-cpu-demo-limit.yaml
```

Verify details of the Pod we created. The Pod inherits the CPU Request from namespace as its default and specifies own CPU Limits.
```console
kubectl get pod default-cpu-demo-limit --output=yaml --namespace=default-cpu-example
```
```
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: default-cpu-demo-limit-ctr
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "1"
```


* Scenario 3: Pod with CPU Request onlyl, without CPU Limits

Create Pod with specified CPU Request only. 
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-request
  namespace: default-cpu-example
spec:
  containers:
  - name: default-cpu-demo-request-ctr
    image: nginx
    resources:
      requests:
        cpu: "0.75"
EOF
```

Verify details of the Pod we created. The Pod inherits the CPU Limits from namespace as its default and specifies own CPU Requests.
```console
kubectl get pod default-cpu-demo-request --output=yaml --namespace=default-cpu-example
```
```
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: default-cpu-demo-request-ctr
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: 750m
```



