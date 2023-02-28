# Persistence

!!! Scenario
    * Creat Pod with `emptyDir` type Volume. Container in the Pod will mount default directory `/var/lib/kubelet/pods/` on running node.
    * Create Deployment `Deployment` with `hostPath` type volume. Container in the Deployment will mount directory defined in `hostPath:` on running node.
    * PV and PVC.
        * Set up NFS Server and share folder `/nfsdata/`.
        * Create PV `mysql-pv` to link to the share folder `/nfsdata/` and set StorageClassName `nfs`.
        * Create PVC `mysql-pvc` mapped with StorageClassName `nfs`.
        * Create Deployment `mysql` to consume PVC `mysql-pvc`.
    * StorageClass
        * Create ServiceAccount `nfs-client-provisioner`.
        * Create ClusterRole `nfs-client-provisioner-runner` and Role `leader-locking-nfs-client-provisioner` and bind them to the ServiceAccount so the     ServiceAccount has authorization to operate the Deployment created in next step.
        * Create Deployment `nfs-client-provisioner` to to add connection information for your NFS server, e.g, `PROVISIONER_NAME` is `k8s-sigs.io/    nfs-subdir-external-provisioner`
        * Create StorageClass `nfs-client` link to `provisioner: k8s-sigs.io/nfs-subdir-external-provisioner`. Releated PV is created automatically.
        * Create PVC `nfs-pvc-from-sc` mapped to PV and StorageClass `nfs-client`.
    * Configuration
        * Create a ConfigMap for content of a file, and mount this ConfigMap to a specific file in a Pod.
        * Create a ConfigMap for username and password, and consume them within a Pod.
        * Use ConfigMap as environment variables in Pod. 



!!! Tips
    * Delete PVC first, then delete PV.
    * If facing `Terminating` status when delete a PVC, use `kubectl edit pvc <your_pvc_name>` and remove `finalize: <your_value>`.


## emptyDir

Create a Pod `hello-producer` with `emptyDir` type Volume.
```console
cat > pod-emptydir.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
 name: hello-producer
spec:
 containers:
 - image: busybox
   name: producer
   volumeMounts:
   - mountPath: /producer_dir
     name: shared-volume
   args:
   - /bin/sh
   - -c
   - echo "hello world" > /producer_dir/hello; sleep 30000
 volumes:
 - name: shared-volume
   emptyDir: {}
EOF


kubectl apply -f pod-emptydir.yaml
```

The Pod `hello-producer` is running on node `cka003`. 
```console
kubectl get pod hello-producer -owide
```
The Pod is running on node `cka003`.
```
NAME             READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
hello-producer   1/1     Running   0          6s    10.244.102.24   cka003   <none>           <none>
```

Log onto `cka003` because the Pod `hello-producer` is running on the node.

Set up the environment `CONTAINER_RUNTIME_ENDPOINT` for command `crictl`. Suggest to do the same for all nodes.
```console
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
```

Run command `crictl ps` to get the container ID of Pod `hello-producer`.
```console
crictl ps |grep hello-producer
```

The ID of container `producer` is `05f5e1bb6a1bb`.
```
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
50058afb3cba5       62aedd01bd852       About an hour ago   Running             producer            0                   e6953bd4833a7       hello-producer
```

Run command `crictl inspect` to get the path of mounted `shared-volume`, which is `emptyDir`.
```console
crictl inspect 50058afb3cba5 | grep source | grep empty
```
The result is below.
```
"source": "/var/lib/kubelet/pods/d7424f86-534a-48f9-9001-9d2a6e822b12/volumes/kubernetes.io~empty-dir/shared-volume",
```

Change the path to the path of mounted `shared-volume` we get above. We will see the content `hello world` of file `hello`. 
```console
cd /var/lib/kubelet/pods/d7424f86-534a-48f9-9001-9d2a6e822b12/volumes/kubernetes.io~empty-dir/shared-volume
cat hello
```

The path `/producer_dir` inside the Pod is mounted to the local host path `/var/lib/kubelet/pods/d7424f86-534a-48f9-9001-9d2a6e822b12/volumes/kubernetes.io~empty-dir/shared-volume`.

The file `/producer_dir/hello` we created inside the Pod is actually in the host local path.

Let's delete the container `producer`, the container `producer` will be started again with new container ID and the file `hello` is still there.
```console
crictl ps
crictl stop <your_container_id>
crictl rm <your_container_id>
```

Let's delete the Pod `hello-producer`, the container `producer` is deleted, file `hello` is deleted.
```console
kubectl delete pod hello-producer 
```



## hostPath

Apply below yaml file to create a MySQL Pod and mount a `hostPath`.
It'll mount host directory `/tmp/mysql` to Pod directory `/var/lib/mysql`.
Check locally if directory `/tmp/mysql` is in place. If not, create it using `mkdir /tmp/mysql`.
```console
cat > mysql-hostpath.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql
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
       volumeMounts:
       - name: mysql-vol
         mountPath: /var/lib/mysql
     volumes:
     - hostPath:
         path: /tmp/mysql
       name: mysql-vol
EOF

kubectl apply -f mysql-hostpath.yaml
```


Verify MySQL Availability

Check status of MySQL Pod. Need document the Pod name and node it's running on.
```console
kubectl get pod -l app=mysql -o wide
```
Result
```
NAME                     READY   STATUS              RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
mysql-749c8ddd67-h2rgs   0/1     ContainerCreating   0          28s   <none>   cka003   <none>           <none>
```

Attach into the MySQL Pod on the running node.
```console
kubectl exec -it <your_pod_name> -- bash
```

Within the Pod, go to directory `/var/lib/mysql`, all files in the directory are same with all files in directory `/tmp/mysql` on node `cka003`.

Connect to the database in the Pod.
```console
mysql -h 127.0.0.1 -uroot -ppassword
```

Operate the database.
```
mysql> show databases;
mysql> connect mysql;
mysql> show tables;
mysql> exit
```



## PV and PVC

Here we will use NFS as backend storage to demo how to deploy PV and PVC.

### Set up NFS Share

1. Install nfs-kernel-server

Log onto `cka002`.

Choose one Worker `cka002` to build NFS server. 
```console
sudo apt-get install -y nfs-kernel-server
```

2. Configure Share Folder

Create share folder.  
```console
mkdir /nfsdata
```

Append one line in file `/etc/exports`.
```console
cat >> /etc/exports << EOF
/nfsdata *(rw,sync,no_root_squash)
EOF
```

There are many different NFS sharing options, including these:

* `*`: accessable to all IPs, or specific IPs.
* `rw`: Share as read-write. Keep in mind that normal Linux permissions still apply. (Note that this is a default option.)
* `ro`: Share as read-only.
* `sync`: File data changes are made to disk immediately, which has an impact on performance, but is less likely to result in data loss. On som* `distributions this is the default.
* `async`: The opposite of sync; file data changes are made initially to memory. This speeds up performance but is more likely to result in data loss. O* `some distributions this is the default.
* `root_squash`: Map the root user and group account from the NFS client to the anonymous accounts, typically either the nobody account or the nfsnobod* `account. See the next section, “User ID Mapping,” for more details. (Note that this is a default option.)
* `no_root_squash`: Map the root user and group account from the NFS client to the local root and group accounts.


We will use password-free remote mount based on `nfs` and `rpcbind` services between Linux servers, not based on `smb` service. 
The two servers must first grant credit, install and set up nfs and rpcbind services on the server side, set the common directory, start the service, and mount it on the client

Start `rpcbind` service.
```console
sudo systemctl enable rpcbind
sudo systemctl restart rpcbind
```

Start `nfs` service.
```console
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

Once `/etc/exports` is changed, we need run below command to make change effected.
```console
exportfs -ra
```
Result
```
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "*:/nfsdata".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x
```

Check whether sharefolder is configured. 
```console
showmount -e
```
And see below output.
```
Export list for cka002:
/nfsdata *
```



3. Install NFS Client

Install NFS client on all nodes.
```console
sudo apt-get install -y nfs-common
```



4. Verify NFS Server

Log onto any nodes to verify NFS service and sharefolder list.

Log onto `cka001` and check sharefolder status on `cka002`.
```console
showmount -e cka002
```
Below result will be shown if no issues.
```
Export list for cka002:
/nfsdata *
```



5. Mount NFS

Execute below command to mount remote NFS folder on any other non-NFS-server node, e.g., `cka001` or `cka003`.
```console
mkdir /remote-nfs-dir
mount -t nfs cka002:/nfsdata /remote-nfs-dir/
```

Use command `df -h` to verify mount point. Below is the sample output.
```
Filesystem       Size  Used Avail Use% Mounted on
cka002:/nfsdata   40G  5.8G   32G  16% /remote-nfs-dir
```



### Create PV

Create a PV `mysql-pv`. 
Replace the NFS Server IP with actual IP (here is `<cka002_ip>`) that NFS server `cka002` is running on.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
 name: mysql-pv
spec:
 accessModes:
     - ReadWriteOnce
 capacity:
   storage: 1Gi
 persistentVolumeReclaimPolicy: Retain
 storageClassName: nfs
 nfs:
   path: /nfsdata/
   server: <cka002_ip>
EOF
```

Check the PV.
```console
kubectl get pv
```
The result:
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mysql-pv   1Gi        RWO            Retain           Available           nfs                     19s
```


### Create PVC

Create a PVC `mysql-pvc` and specify storage size, access mode, and storage class. 
The PVC `mysql-pvc` will be binded with PV automatically via storage class name. 
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
EOF
```



### Consume PVC

Update the Deployment `mysql` to consume the PVC created.
```console
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mysql
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
       volumeMounts:
       - name: mysql-persistent-storage
         mountPath: /var/lib/mysql
         subPath: mysqldata
     volumes:
     - name: mysql-persistent-storage
       persistentVolumeClaim:
        claimName: mysql-pvc
EOF
```

Now we can see MySQL files were moved to directory `/nfsdata` on `cka002`




## StorageClass

### Configure RBAC Authorization

RBAC authorization uses the rbac.authorization.k8s.io API group to drive authorization decisions, allowing you to dynamically configure policies through the Kubernetes API.

* ServiceAccount: `nfs-client-provisioner`
* namespace: `dev`

* ClusterRole: `nfs-client-provisioner-runner`. Grant authorization on node, pv, pvc, sc, event.
* ClusterRoleBinding: `run-nfs-client-provisioner`, bind above ClusterRole to above ServiceAccount.

* Role: `leader-locking-nfs-client-provisioner`. Grant authorization on endpoint.
* RoleBinding: `leader-locking-nfs-client-provisioner`, bind above Role to above ServiceAccount.


Create RBAC Authorization.
```console
cat > nfs-provisioner-rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: dev
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: dev
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: dev
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF


kubectl apply -f nfs-provisioner-rbac.yaml
```


### Create Provisioner's Deloyment

Create Deloyment `nfs-client-provisioner` by consuming volume `nfs-client-root` mapped to `/nfsdata` on `<cka002_ip>`(`cka002`). 
Replace NFS server IP with actual IP (here is `<cka002_ip>`)
```console
cat > nfs-provisioner-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: liyinlin/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: <cka002_ip>
            - name: NFS_PATH
              value: /nfsdata
      volumes:
        - name: nfs-client-root
          nfs:
            server: <cka002_ip>
            path: /nfsdata
EOF

kubectl apply -f nfs-provisioner-deployment.yaml
```


### Create NFS StorageClass

Create StorageClass `nfs-client`. Define the NFS subdir external provisioner's Kubernetes Storage Class.
```console
vi nfs-storageclass.yaml
```
And add below info to create NFS StorageClass.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  pathPattern: "${.PVC.namespace}/${.PVC.annotations.nfs.io/storage-path}"
  onDelete: delete
```

Apply the yaml file.
```console
kubectl apply -f nfs-storageclass.yaml
```


### Create PVC

Create PVC `nfs-pvc-from-sc`.
```console
kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-pvc-from-sc
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

Check the PVC status we ceated.
```console
kubectl get pvc nfs-pvc-from-sc
```
The status is `Pending`.
```
NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-from-sc   Pending                                      nfs-client     112s
```

Check pending reason
```console
kubectl describe pvc nfs-pvc-from-sc
```
It's pending on waiting for a volume to be created.
```
Events:
  Type    Reason                Age               From                         Message
  ----    ------                ----              ----                         -------
  Normal  ExternalProvisioning  9s (x6 over 84s)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "k8s-sigs.io/nfs-subdir-external-provisioner" or manually created by system administrator
```




### Consume PVC

Create Deployment `mysql-with-sc-pvc` to consume the PVC `nfs-pvc-from-sc`.
```console
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-with-sc-pvc
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
        volumeMounts:
        - name: mysql-pv
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-pv
        persistentVolumeClaim:
          claimName: nfs-pvc-from-sc
EOF
```


Check the Deployment status.
```console
kubectl get deployment mysql-with-sc-pvc -o wide
```
Result
```
NAME                READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES      SELECTOR
mysql-with-sc-pvc   1/1     1            1           16s   mysql        mysql:8.0   app=mysql
```

With the comsumption from Deployment `mysql-with-sc-pvc`, the status of PVC `nfs-pvc-from-sc` is now status `Bound` from `Pending`.
```console
kubectl get pvc nfs-pvc-from-sc
```
Result
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-from-sc   Bound    pvc-edf70dff-7407-4b38-aac9-9c2dd6a84316   1Gi        RWX            nfs-client     52m
```

Check related Pods status. Be noted that the Pod `mysql-with-sc-pvc-7c97d875f8-dwfkc` is running on `cka003`.
```console
kubectl get pod -o wide -l app=mysql
```
Result
```
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
mysql-774db46945-h82kk               1/1     Running   0          69m     10.244.112.26   cka002   <none>           <none>
mysql-with-sc-pvc-7c97d875f8-wkvr9   1/1     Running   0          3m37s   10.244.102.27   cka003   <none>           <none>
```

Let's check directory `/nfsdata/` on NFS server `cka002`. 
```console
ll /nfsdata/
```
Two folders were created. Same content of `/remote-nfs-dir/` on other nodes.
```
drwxrwxrwx  6 systemd-coredump root 4096 Jul 23 23:35 dev/
drwxr-xr-x  6 systemd-coredump root 4096 Jul 23 22:29 mysqldata/
```

Namespace name is used as folder name under directory `/nfsdata/` and it is mounted to Pod.
By default, namespace name will be used at mount point. 
If we want to use customized folder for that purpose, we need claim an annotation `nfs.io/storage-path`, e.g., below example.

Create PVC test-claim on Namespace `kube-system` and consume volume `nfs-client`.
```console
kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  namespace: kube-system
  annotations:
    nfs.io/storage-path: "test-path"
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

In above case, the PVC was created in `kube-system` Namespace, hence we can see directory `test-path` is under directory`kube-system` on node `cka002`. 

Overall directory structure of folder `/nfsdata/` looks like below.
```console
tree -L 1 /nfsdata/ 
```
Result
```
/nfsdata/
├── dev
├── kube-system
└── mysqldata
```

Please be noted that above rule is following `nfs-subdir-external-provisioner` implementation. It's may be different with other `provisioner`.

Detail about `nfs-subdir-external-provisioner` project is [here](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)



## Configuration

### ConfigMap

Create ConfigMap `cm-nginx` to define content of `nginx.conf`.
```console
vi configmap.yaml
```
Paste below content.
```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    cattle.io/creator: norman
  name: cm-nginx
  namespace: dev
data:
  nginx.conf: |-
    user  nginx;
    worker_processes  2;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        include /etc/nginx/conf.d/*.conf;
    }
```

Apply the ConfigMap.
```console
kubectl apply -f configmap.yaml
```

Create Pod `nginx-with-cm`.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-cm
spec:
 containers:
 - name: nginx
   image: nginx
   volumeMounts:
   - name: foo
     mountPath: "/etc/nginx/nginx.conf"
     subPath:  nginx.conf
 volumes:
 - name: foo
   configMap:
     name: cm-nginx
EOF
```

!!! Note
    * By default to mount ConfigMap, Kubernetes will overwrite all content of the mount point. We can use `volumeMounts.subPath` to specify that only overwrite the file `nginx.conf` defined in `mountPath`.
    * Is we use `volumeMounts.subPath` to mount a Container Volume, Kubernetes won't do hot update to reflect real-time update.


Verify if the `nginx.conf` mounted from outside is in the Container by comparing with above file.
```console
kubectl exec -it nginx-with-cm -- sh 
cat /etc/nginx/nginx.conf
```




### Secret

Encode password with base64  
```console
echo -n admin | base64  
YWRtaW4=

echo -n 123456 | base64
MTIzNDU2
```

Create Secret `mysecret`.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
data:
  username: YWRtaW4=
  password: MTIzNDU2
EOF
```

Using Volume to mount (injection) Secret to a Pod.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox-with-secret
spec:
 containers:
 - name: mypod
   image: busybox
   args:
    - /bin/sh
    - -c
    - sleep 1000000;
   volumeMounts:
   - name: foo
     mountPath: "/tmp/secret"
 volumes:
 - name: foo
   secret:
    secretName: mysecret
EOF
```

Let's attach to the Pod `busybox-with-secret` to verify if two data elements of `mysecret` are successfully mounted to the path `/tmp/secret` within the Pod.
```console
kubectl exec -it busybox-with-secret -- sh
```
By executing below command, we can see two data elements are in the directory `/tmp/secret` as file type. 
```console
/ # ls -l /tmp/secret/
lrwxrwxrwx    1 root     root            15 Jul 23 16:30 password -> ..data/password
lrwxrwxrwx    1 root     root            15 Jul 23 16:30 username -> ..data/username
```
And we can get the content of each element, which are predefined before.
```console
/ # cat /tmp/secret/username
admin

/ # cat /tmp/secret/password
123456
```



### Additional Cases

#### Various way to create ConfigMap

ConfigMap can be created by file, directory, or value. 

Let's create a ConfigMap `colors` includes:

* Four files with four color names.
* One file with favorite color name.

```console
mkdir configmap
cd configmap
mkdir primary

echo c > primary/cyan
echo m > primary/magenta
echo y > primary/yellow
echo k > primary/black
echo "known as key" >> primary/black
echo blue > favorite
```
Final structure looks like below via command `tree configmap`.
```
configmap
├── favorite
└── primary
    ├── black
    ├── cyan
    ├── magenta
    └── yellow
```

Create ConfigMap referring above files we created. Make sure we are now in the path `~/configmap`.
```console
kubectl create configmap colors \
--from-literal=text=black  \
--from-file=./favorite  \
--from-file=./primary/
```

Check content of the ConfigMap `colors`.
```console
kubectl get configmap colors -o yaml
```
```
apiVersion: v1
data:
  black: |
    k
    known as key
  cyan: |
    c
  favorite: |
    blue
  magenta: |
    m
  text: black
  yellow: |
    y
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-12T16:38:27Z"
  name: colors
  namespace: dev
  resourceVersion: "2377258"
  uid: d5ab133f-5e4d-41d4-bc9e-2bbb22a872a1
```




#### Set environment variable via ConfigMap

Here we will create a Pod `pod-configmap-env` and set the environment variable `ilike` and assign value of `favorite` from ConfigMap `colors`.
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-env
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: ilike
      valueFrom:
        configMapKeyRef:
          name: colors
          key: favorite
EOF
```

Attach to the Pod `pod-configmap-env`.
```console
kubectl exec -it pod-configmap-env -- bash
```

Verify the value of env variable `ilike` is `blue`, which is the value of `favorite` of ConfigMap `colors`.
```console
root@pod-configmap-env:/# echo $ilike
blue
```

We can also use all key-value of ConfigMap to set up environment variables of Pod.
```console
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
 name: pod-configmap-env-2
spec:
 containers:
 - name: nginx
   image: nginx
   envFrom:
    - configMapRef:
        name: colors
EOF
```

Attach to the Pod `pod-configmap-env-2`.
```console
kubectl exec -it pod-configmap-env-2 -- bash
```

Verify the value of env variables based on key-values we defined in ConfigMap `colors`.
```console
root@pod-configmap-env-2:/# echo $black
k known as key
root@pod-configmap-env-2:/# echo $cyan
c
root@pod-configmap-env-2:/# echo $favorite
blue
```



