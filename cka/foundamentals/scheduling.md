# Scheduling

!!! Scenario
    * Configure nodeSelector for Pod.
    * Configure nodeName for Node.
    * Use `podAffinity` to group Pods.
    
    * Taints & Tolerations
        * Set Taints
        * Set Tolerations
        * Remove Taints



## nodeSelector

Let's assume the scenario below.

* We have a group of high performance servers.
* Some applications require high performance computing.
* These applicaiton need to be scheduled and running on those high performance servers.

We can leverage Kubernetes attributes node `label` and `nodeSelector` to group resources as a whole for scheduling to meet above requirement.




1. Label Node

Let's label `cka002` with `Configuration=hight`.
```console
kubectl label node cka002 configuration=hight
```

Verify. We wil see the label `configuration=hight` on `cka002`.
```console
kubectl get node --show-labels
```


2. Configure nodeSelector for Pod

Create a Pod and use `nodeSelector` to schedule the Pod running on specified node.
```console
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql-nodeselector
spec:
 selector:
   matchLabels:
     app: mysql
 template:
   metadata:
     labels:
       app: mysql
   spec:
     containers:
     - image: mysql:8.0
       name: mysql
       env:
       - name: MYSQL_ROOT_PASSWORD
         value: password
       ports:
       - containerPort: 3306
         name: mysql
     nodeSelector:
       configuration: hight
EOF
```

Let's check with node the Pod `mysql-nodeselector` is running.
```console
kubectl get pod -l app=mysql -o wide |  grep mysql-nodeselector
```

With below result, Pod `mysql-nodeselector` is running on `cka002` node.
```
NAME                                  READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
mysql-nodeselector-6b7d9c875d-vs8mk   1/1     Running   0          7s     10.244.112.29   cka002   <none>           <none>
```



## nodeName

Be noted, `nodeName` has hightest priority as it's not scheduled by `Scheduler`.

Create a Pod `nginx-nodename` with `nodeName=cka003`.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nodename
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: cka003
EOF
```

Verify if Pod `nginx-nodename` is running on `ckc003 node.
```console
kubectl get pod -owide |grep nginx-nodename
```
Result
```
NAME                                      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
nginx-nodename                            1/1     Running   0          8s     10.244.102.29   cka003   <none>           <none>
```




## Affinity

In Kubernetes cluster, some Pods have frequent interaction with other Pods. With that situation, it's suggested to schedule these Pods running on same node. 
For example, Two Pods Nginx and Mysql, we need deploy them on one node if they frequently communicate.

We can use `podAffinity` to select Pods based on their relationship. 

There are two scheduling type of `podAffinity`.

* `requiredDuringSchedulingIgnoredDuringExecution`(硬亲和)
* `preferredDuringSchedulingIgnoredDuringExecution`(软亲和)

`topologyKey` could be set by below types:

* `kubernetes.io/hostname` ＃NodeName
* `failure-domain.beta.kubernetes.io/zone` ＃Zone 
* `failure-domain.beta.kubernetes.io/region` # Region 

We can set node Label to classify Name/Zone/Region of node, which can be used by `podAffinity`.

Create a Pod Nginx.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

Create a Pod MySql.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql
    env:
     - name: "MYSQL_ROOT_PASSWORD"
       value: "123456"
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        topologyKey: kubernetes.io/hostname
EOF
```

As we configured `podAffinity`, so Pod `mysql` will be scheduled to the same node with Pod `nginx` by Label `app:nginx`.

Via the command `kubectl get pod -o wide` we can get two Pods are running on node `cka002`.









## Taints & Tolerations

### Concept

Node affinity is a property of Pods that attracts them to a set of nodes (either as a preference or a hard requirement). 
Taints are the opposite -- they allow a node to repel a set of pods.

Tolerations are applied to pods. 
Tolerations allow the scheduler to schedule pods with matching taints. 
Tolerations allow scheduling but don't guarantee scheduling: the scheduler also evaluates other parameters as part of its function.

Taints and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes. 
One or more taints are applied to a node; this marks that the node should not accept any pods that do not tolerate the taints.

In the production environment, we generally configure Taints for Control Plane nodes (in fact, most K8s installation tools automatically add Taints to Control Plane nodes), because Control Plane only runs Kubernetes system components. 
If it is used to run application Pods, it is easy to consume resources. In the end, the Control Plane node will crash. 
If we need to add additional system components to the Control Plane node later, we can configure Tolerations for the corresponding system component Pod to tolerate taints.


### Set Taints

Set `cka003` node to taint node. Set status to `NoSchedule`, which won't impact existing Pods running on `cka003`.
```console
kubectl taint nodes cka003 key=value:NoSchedule
```

### Set Tolerations

We can use Tolerations to let Pods schedule to a taint node. 
```console
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql-tolerations
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
      tolerations:
      - key: "key"
        operator: "Equal"
        value: "value"
        effect: "NoSchedule"
EOF
```

The Pod of Deployment `mysql-tolerations` is scheduled and running on node `cka003` with `tolerations` setting, which is a taint node.
```console
kubectl get pod -o wide | grep mysql-tolerations
```





### Remove Taints

```console
kubectl taint nodes cka003 key-
```


