# Service

!!! Scenario
    * Create Deployment `httpd-app`.
    * Create Service `httpd-app` with type `ClusterIP`, which is default type and accessable internally.
    * Verify the access to Pod IP and Service ClusterIP.
    * Update Service `httpd-app` with type `NodePort`. No change to the Deployment `httpd-app`.
    * Verify the access to Node. The access will route to Pod. The service is now accesable from outside.
    * Create Headless Service `web` and StatefulSet `web`.
    * Service Internal Traffic Policy


## ClusterIP

### Create Service

Create a Deployment `http-app`.
Create a Service `httpd-app` link to Development `http-app` by Label Selector. 

Service type is `ClusterIP`, which is default type and accessable internally. 

```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpd-app
spec:
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: httpd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-app
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 2
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80
EOF
```

Execute command `kubectl get deployment,service,pod -o wide` to check resources we created. 
```
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/httpd-app   0/2     2            0           14s   httpd        httpd    app=httpd

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/httpd-app   ClusterIP   11.244.247.7   <none>        80/TCP    14s   app=httpd

NAME                             READY   STATUS    RESTARTS   AGE    IP              NODE     NOMINATED NODE   READINESS GATES
pod/httpd-app-6496d888c9-4nb6z   1/1     Running   0          77s    10.244.102.21   cka003   <none>           <none>
pod/httpd-app-6496d888c9-b7xht   1/1     Running   0          77s    10.244.112.19   cka002   <none>           <none>
```

Verify the access to Pod IPs.
```console
curl 10.244.102.21
curl 10.244.112.19
```
And receive below successful information.
```
<html><body><h1>It works!</h1></body></html>
```

Verify the access via ClusterIP with Port.
```console
curl 11.244.247.7:80
```
And receive below successful information.
```
<html><body><h1>It works!</h1></body></html>
```



### Expose Service

Create and attach to a temporary Pod `nslookup` and to verify DNS resolution. The option `--rm` means delete the Pod after exit.
```console
kubectl run -it nslookup --rm --image=busybox:1.28
```

After attach to the Pod, run command `nslookup httpd-app`. We receive the ClusterIP of Service `httpd-app` and full domain name.
```console
/ # nslookup httpd-app
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      httpd-app
Address 1: 11.244.247.7 httpd-app.dev.svc.cluster.local
```

We can check the IP of temporary Pod `nslookup` in a new terminal by executing command `kubectl get pod -o wide`. 
The Pod `nslookup` has Pod IP `10.244.112.20`.
```console
kubectl get pod nslookup
```
Result
```
NAME       READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
nslookup   1/1     Running   0          2m44s   10.244.112.20   cka002   <none>           <none>
```



## NodePort

Create and apply yaml file `svc-nodeport.yaml` to create a Service `httpd-app`.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpd-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  selector:
     app: httpd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-app
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 2
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80
EOF
```

We will receive below output. The command `kubectl apply -f <yaml_file>` will update configuration to existing resources.
Here the Service `httpd-app` is changed from `ClusterIP` to `NodePort` type. No change to the Deployment `httpd-app`.
```
service/httpd-app configured
deployment.apps/httpd-app unchanged
```

Check the Service `httpd-app` via `kubectl get svc`. 
IP is the same.
Type is changed to NodePort.
Port numbers is changed from `80/TCP` to `80:30080/TCP`.
```
NAME        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
httpd-app   NodePort   11.244.247.7   <none>        80:30080/TCP   18m
```

Test the connection to the Service `httpd-app` via command `curl <your_node_ip>:30080` to each node.
```console
curl <node1_ip>:30080
curl <node2_ip>:30080
curl <node3_ip>:30080
```
We will receive below successful information.
```
<html><body><h1>It works!</h1></body></html>
```





## Headless Service

Create Headless Service `web` and StatefulSet `web`.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
          name: web
EOF
```

Check Pos by command `kubectl get pod -owide -l app=web`.
```
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
web-0   1/1     Running   0          9s    10.244.102.22   cka003   <none>           <none>
web-1   1/1     Running   0          6s    10.244.112.21   cka002   <none>           <none>
```

Get details of the Service by command `kubectl describe svc -l app=web`.
```
Name:              web
Namespace:         dev
Labels:            app=web
Annotations:       <none>
Selector:          app=web
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              web  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.102.22:80,10.244.112.21:80
Session Affinity:  None
Events:            <none>
```

Attach to the temporary Pod `nslookup` and use `nslookup` to verify DNS resolution.
```console
kubectl run -it nslookup --rm --image=busybox:1.28
```

With `nslookup` command for Headless Service `web`, we received two Pod IPs, not ClusterIP due to Headless Service. 
```console
/ # nslookup web
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web
Address 1: 10.244.112.21 web-1.web.dev.svc.cluster.local
Address 2: 10.244.102.22 web-0.web.dev.svc.cluster.local
```

We can also use `nslookup` for `web-0.web` and `web-1.web`. Every Pod of Headless Service has own Service Name for DNS lookup.
```console
/ # nslookup web-0.web
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.web
Address 1: 10.244.102.22 web-0.web.dev.svc.cluster.local

/ # nslookup web-1.web
Server:    11.244.0.10
Address 1: 11.244.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.web
Address 1: 10.244.112.21 web-1.web.dev.svc.cluster.local
```

Clean up.
```console
kubectl delete sts web
kubectl delete service httpd-app web
kubectl delete deployment httpd-app 
```



## Service Internal Traffic Policy

!!! Scenario 
     * Simulate how Service Internal Traffic Policy works.
     * Expected result:
         * With setting Service `internalTrafficPolicy: Local`, the Service only route internal traffic within the nodes that Pods are running. 

!!! Backgroud
    * Service Internal Traffic Policy enables internal traffic restrictions to only route internal traffic to endpoints within the node the traffic originated from. 
    * The "internal" traffic here refers to traffic originated from Pods in the current cluster.
    * By setting its `.spec.internalTrafficPolicy` to Local. This tells kube-proxy to only use node local endpoints for cluster internal traffic.
    * For pods on nodes with no endpoints for a given Service, the Service behaves as if it has zero endpoints (for Pods on this node) even if the service does have endpoints on other nodes.

Demo:

Create Deployment `my-nginx` and Service `my-nginx`.
```console
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
```

With command `kubectl get pod -o wide`, we know the Pod of Deployment `my-nginx` is running on node `cka003`.
```
NAME                                      READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
my-nginx-cf54cdbf7-bscf8                  1/1     Running   0          9h      10.244.112.63   cka002   <none>           <none>
```

Let's send http request from `cka001` to the Pod on `cka002`. 
We will receive `Welcome to nginx!` information, which means the Pod is accessable from other nodes.
```console
curl 11.244.163.60
```

Let's modify the Serivce `my-nginx` and specify `internalTrafficPolicy: Local`. 
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
  internalTrafficPolicy: Local
EOF
```

Let's send http request from `cka001` to the http request to the Pod again. 
We will receive `curl: (7) Failed to connect to 11.244.163.60 port 80: Connection refused` error information.
```console
curl 11.244.163.60
```

Let's log onto `cka002` and the http request to the Pod again. 
We will receive `Welcome to nginx!` information, 
```console
curl 11.244.163.60
```

!!! Conclution
    With setting Service `internalTrafficPolicy: Local`, the Service only route internal traffic within the nodes that Pods are running. 




!!! Scenario
    * Create a `nginx` deployment
    * Add port number and alias name of the `nginx` Pod.
    * Expose the deployment with internal traffic to local only.


Demo:

Create deployment `my-nginx` with port number `80`.
```console
kubectl create deployment my-nginx --image=nginx --port=80
```

Edit deployment.
```console
kubectl edit deployment my-nginx
```
Add port alias name `http`.
Refer to the link for deployment yaml template https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
```
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
          name: http
```

Expose the deployment with `NodePort` type.
```console
kubectl expose deployment my-nginx --port=80 --target-port=http --name=my-nginx-svc --type=NodePort
```

Edit the service. Change `internalTrafficPolicy` from `Cluster` to `Local`.
```console
kubectl edit svc my-nginx-svc 
```

Verify the access. Note, the pod is running on node `cka003`. We will see below expected results.
```console
curl <deployment_pod_ip>:80    # succeed on node cka003. internalTrafficPolicy is effective.
curl <service_cluster_ip>:80   # succeed on all nodes.
curl <node_ip>:<ext_port>      # succeed on all nodes.
```






