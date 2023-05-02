# 案例分析：Kubernetes资源常见操作

演示场景：

* 节点标签（Node Label）
* 注解（Annotation）
* 命名空间（Namespace）
* ServiceAccount 授权（ServiceAccount Authorization）
  * 授权默认 ServiceAccount 访问 API
* 部署（Deployment）
* 暴露服务（Expose Service）
* 扩展部署（Scale out the Deployment）
* 滚动升级（Rolling update）
* 回滚升级（Rolling back update）
* 事件（Event）
* 日志记录（Logging）

## Node Label

* Add/update/remove node Label.

```console
# Update node label
kubectl label node cka002 node=demonode

# Get node info with label info
kubectl get node --show-labels

# Search node by label
kubectl get node -l node=demonode

# Remove a lable of node
kubectl label node cka002 node-
```

## Annotation

Create Nginx deployment

```console
kubectl create deploy nginx --image=nginx:mainline
```

Get Annotation info.

```console
kubectl describe deployment/nginx
```

Result

```
......
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
......
```

Add new Annotation.

```console
kubectl annotate deployment nginx owner=James.H
```

Now annotation looks like below.

```
......
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
                        owner: James.H
Selector:               app=nginx
......
```

Update/Overwrite Annotation.

```console
kubectl annotate deployment/nginx owner=K8s --overwrite
```

Now annotation looks like below.

```
......
Annotations:            deployment.kubernetes.io/revision: 1
                        owner: K8s
Selector:               app=nginx
......
```

Remove Annotation

```console
kubectl annotate deployment/nginx owner-
```

```
......
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
......
```

Clean up

```console
kubectl delete deployment nginx
```

## Namespace

* Get current available namespaces.

```console
kubectl get namespace
```

Result

```
NAME              STATUS   AGE
default           Active   3h45m
dev               Active   3h11m
kube-node-lease   Active   3h45m
kube-public       Active   3h45m
kube-system       Active   3h45m
```

* Get Pod under a specific namespace.

```console
kubectl get pod -n kube-system
```

Result

```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5c64b68895-jr4nl   1/1     Running   0          3h25m
calico-node-dsx76                          1/1     Running   0          3h25m
calico-node-p5rf2                          1/1     Running   0          3h25m
calico-node-tr22l                          1/1     Running   0          3h25m
coredns-6d8c4cb4d-g4jxc                    1/1     Running   0          3h45m
coredns-6d8c4cb4d-sqcvj                    1/1     Running   0          3h45m
etcd-cka001                                1/1     Running   0          3h45m
kube-apiserver-cka001                      1/1     Running   0          3h45m
kube-controller-manager-cka001             1/1     Running   0          3h45m
kube-proxy-5cdbj                           1/1     Running   0          3h41m
kube-proxy-cm4hc                           1/1     Running   0          3h45m
kube-proxy-g4w52                           1/1     Running   0          3h41m
kube-scheduler-cka001                      1/1     Running   0          3h45m
```

* Get Pods in all namespaces.

```console
kubectl get pod --all-namespaces
kubectl get pod -A
```

## ServiceAccount Authorization

With Kubernetes 1.23 and lower version, when we create a new namespace, Kubernetes will automatically create a ServiceAccount `default` and a token `default-token-xxxxx`.

With Kubernetes 1.24, only ServiceAccount `default` is created automatically when a new namespace is created, need manually create a toke linked to the ServiceAccount `default`.

Here is an example to create a new namespace `dev`, we can see that only ServiceAcccount: `default` was created in namespace `dev`, no secretes (token) linked to the ServiceAccount `default`.

```console
kubectl create namespace dev
kubectl get serviceaccount -n dev
kubectl get secrets -n dev
```

There is a default cluster role `admin`.
But there is no clusterrole binding to the cluster role `admin`.

```console
kubectl get clusterrole admin
kubectl get clusterrolebinding | grep ClusterRole/admin
```

Role and rolebinding is namespaces based. On namespace `dev`, there is no role and rolebinding.

```console
kubectl get role -n dev
kubectl get rolebinding -n dev
```

A Secret in the Kubernetes cluster is an object and it is used to store sensitive information such as username, password, and token, etc.
The objective of Secrets is to encode or hash the credentials.
The secrets can be reused in the various Pod definition file.

A `kubernetes.io/service-account-token` type of Secret is used to store a token that identifies a service account.
When using this Secret type, you need to ensure that the `kubernetes.io/service-account.name` annotation is set to an existing service account name.

Let's create token for the ServiceAcccount: `default` in namespace `dev`.

```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Secret
metadata:
  name: default-token-dev
  namespace: dev
  annotations:
    kubernetes.io/service-account.name: "default"
type: kubernetes.io/service-account-token

EOF
```

Now we get ServiceAcccount: `default` and Secret (token) `default-token-dev` in namespace `dev`.

```console
kubectl get serviceaccount -n dev
kubectl get secrets -n dev
```

Get token of the service account `default`.

```console
TOKEN=$(kubectl -n dev describe secret $(kubectl -n dev get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d ' ')
echo $TOKEN
```

Get API Service address.

```console
APISERVER=$(kubectl config view | grep https | cut -f 2- -d ":" | tr -d " ")
echo $APISERVER
```

Get Pod resources in namespace `dev` via API server with JSON layout.

```console
curl $APISERVER/api/v1/namespaces/dev/pods --header "Authorization: Bearer $TOKEN" --insecure
```

We will receive `403 forbidden` error message. The ServiceAccount `default` does not have authorization to access pod in namespace `dev`.

Let's create a rolebinding `rolebinding-admin` to bind cluster role `admin` to service account `default` in namespapce `dev`.
Hence service account `default` is granted adminstrator authorization in namespace `dev`.

```console
# Usage:
kubectl create rolebinding <rule> --clusterrole=<clusterrule> --serviceaccount=<namespace>:<name> --namespace=<namespace>

# Crate rolebinding:
kubectl create rolebinding rolebinding-admin --clusterrole=admin --serviceaccount=dev:default --namespace=dev
```

Result looks like below by executing `kubectl get rolebinding -n dev`.

```
NAME                ROLE                AGE
rolebinding-admin   ClusterRole/admin   10s
```

Try again, get pod resources in namespace `dev` via API server with JSON layout.

```console
curl $APISERVER/api/v1/namespaces/dev/pods --header "Authorization: Bearer $TOKEN" --insecure
```

Clean up.

```console
kubectl delete namespace dev
```

## Deployment

Create a Ubuntu Pod for operation. And attach to the running Pod.

```console
kubectl create -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
  labels:
    app: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: ["/bin/sleep", "3650d"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF

kubectl exec --stdin --tty ubuntu -- /bin/bash
```

Create a deployment, option `--image` specifies a image，option `--port` specifies port for external access.
A pod is also created when deployment is created.

```console
kubectl create deployment myapp --image=docker.io/jocatalin/kubernetes-bootcamp:v1 --replicas=1 --port=8080
```

Get deployment status

```console
kubectl get deployment myapp -o wide
```

Result

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS            IMAGES                                       SELECTOR
myapp   1/1     1            1           79s   kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v1   app=myapp
```

Get detail information of deployment.

```console
kubectl describe deployment myapp
```

Result

```
Name:                   myapp
Namespace:              dev
CreationTimestamp:      Sat, 23 Jul 2022 14:36:43 +0800
Labels:                 app=myapp
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=myapp
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
  Containers:
   kubernetes-bootcamp:
    Image:        docker.io/jocatalin/kubernetes-bootcamp:v1
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   myapp-b5d775f5d (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  95s   deployment-controller  Scaled up replica set myapp-b5d775f5d to 1
```

## Expose Service

Get the Pod and Deployment we created just now.

```console
kubectl get deployment myapp -o wide
kubectl get pod -o wide
```

Result

```
NAME                    READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
myapp-b5d775f5d-cx8dx   1/1     Running   0          2m34s   10.244.102.7   cka003   <none>           <none>
```

Send http request to the Pod `curl 10.244.102.7:8080` with below result.

```
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-6jtgs | v=1
```

To make pod be accessed outside, we need expose port `8080` to a node port. A related service will be created.

```console
kubectl expose deployment myapp --type=NodePort --port=8080
```

Get details of service `myapp` by executing `kubectl get svc myapp -o wide`.

```
NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE   SELECTOR
myapp   NodePort   11.244.74.3   <none>        8080:30514/TCP   7s    app=myapp
```

Send http request to service port.

```console
curl 11.244.74.3:8080
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-cx8dx | v=1
```

Get more details of the service.

```console
kubectl get svc myapp -o yaml
kubectl describe svc myapp
```

Get details of related endpoint `myapp` by executing `kubectl get endpoints myapp -o wide`.

```
NAME    ENDPOINTS           AGE
myapp   10.244.102.7:8080   43s
```

Send http request to the service and node sucessfully. Pod port `8080` is mapped to node port `32566`.

Send http request to node port on `cka003`.

```console
curl <cka003_node_ip>:30514
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-6jtgs | v=1
```

Attach to Ubuntu Pod we created and send http request to the Service and Pod and Node of `myapp`.

```console
kubectl exec --stdin --tty ubuntu -- /bin/bash
curl 10.244.102.7:8080
curl 11.244.74.3:8080
curl <cka003_node_ip>:30514
Hello Kubernetes bootcamp! | Running on: myapp-b5d775f5d-6jtgs | v=1
```

## Scale out Deployment

Scale out by replicaset. We set three replicasets to scale out deployment `myapp`. The number of deployment `myapp` is now three.

```console
kubectl scale deployment myapp --replicas=3
```

Get status of deployment

```console
kubectl get deployment myapp
```

Get status of replicaset

```console
kubectl get replicaset
```

## Rolling update

Command usage: `kubectl set image (-f FILENAME | TYPE NAME) CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N`.

With the command `kubectl get deployment`, we will get deployment name `myapp` and related container name `kubernetes-bootcamp`.

```console
kubectl get deployment myapp -o wide
```

With the command `kubectl set image` to update image to many versions and log the change under deployment's annotations with option `--record`.

```console
kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2 --record
```

Current replicas status

```console
kubectl get replicaset -o wide -l app=myapp
```

Result. Pods are running on new Replicas.

```
NAME               DESIRED   CURRENT   READY   AGE   CONTAINERS            IMAGES                                       SELECTOR
myapp-5dbd68cc99   1         1         0       8s    kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v2   app=myapp,pod-template-hash=5dbd68cc99
myapp-b5d775f5d    3         3         3       14m   kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v1   app=myapp,pod-template-hash=b5d775f5d
```

We can get the change history under `metadata.annotations`.

```console
kubectl get deployment myapp -o yaml
```

Result

```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubernetes.io/change-cause: kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2
      --record=true
  ......
```

We can also get the change history by command `kubectl rollout history`, and show details with specific revision `--revision=<revision_number>`.

```console
kubectl rollout history deployment/myapp
```

Result

```
deployment.apps/myapp 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2 --record=true
```

Get rollout history with specific revision.

```console
kubectl rollout history deployment/myapp --revision=2
```

Roll back to previous revision with command `kubectl rollout undo`, or roll back to specific revision with option `--to-revision=<revision_number>`.

```console
kubectl rollout undo deployment/myapp --to-revision=1
```

Revision `1` was replaced by new revision `3` now.

```console
kubectl rollout history deployment/myapp
```

Result

```
deployment.apps/myapp 
REVISION  CHANGE-CAUSE
2         kubectl set image deployment/myapp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2 --record=true
3         <none>
```

## Event

Get detail event info of related Pod.

```console
kubectl describe pod myapp-b5d775f5d-jlx6g
```

Result looks like below.

```
......
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  54s   default-scheduler  Successfully assigned dev/myapp-b5d775f5d-jlx6g to cka003
  Normal  Pulled     53s   kubelet            Container image "docker.io/jocatalin/kubernetes-bootcamp:v1" already present on machine
  Normal  Created    53s   kubelet            Created container kubernetes-bootcamp
  Normal  Started    53s   kubelet            Started container kubernetes-bootcamp
```

Get detail event info of entire cluster.

```console
kubectl get event
```

## Logging

Get log info of Pod.

```console
kubectl logs -f <pod_name>
kubectl logs -f <pod_name> -c <container_name> 
```

Get a Pod logs

```console
kubectl logs -f myapp-b5d775f5d-jlx6g
```

```
Kubernetes Bootcamp App Started At: 2022-07-23T06:54:18.532Z | Running On:  myapp-b5d775f5d-jlx6g
```

Get log info of K8s components.

```console
kubectl logs kube-apiserver-cka001 -n kube-system
kubectl logs kube-controller-manager-cka001 -n kube-system
kubectl logs kube-scheduler-cka001 -n kube-system
kubectl logs etcd-cka001 -n kube-system
systemctl status kubelet
journalctl -fu kubelet
kubectl logs kube-proxy-5cdbj -n kube-system
```

Clean up.

```console
kubectl delete service myapp
kubectl delete deployment myapp
```
