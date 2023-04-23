# CKA自学笔记17:Persistence

## 摘要

演示场景：

* 创建一个类型为 `emptyDir` 的卷来创建 Pod，Pod 中的容器将会挂载在运行节点上的默认目录 `/var/lib/kubelet/pods/` 中。
* 创建一个类型为 `hostPath` 的卷来创建 Deployment，Deployment 中的容器将会挂载在运行节点上定义的目录 `hostPath:` 中。
* 创建 PV 和 PVC：
  * 设置 NFS 服务器并共享 `/nfsdata/` 目录。
  * 创建 PV `mysql-pv` 并映射到共享目录 `/nfsdata/`，同时设置 StorageClassName 为 `nfs`。
  * 创建 PVC `mysql-pvc` 并映射到 StorageClassName 为 `nfs` 的 PV 上。
  * 创建 Deployment `mysql` 来使用 PVC `mysql-pvc`。
* 创建 StorageClass：
  * 创建 ServiceAccount `nfs-client-provisioner`。
  * 创建 ClusterRole `nfs-client-provisioner-runner` 和 Role `leader-locking-nfs-client-provisioner`，并将其绑定到 ServiceAccount 上，以便该 ServiceAccount 可以操作下一步中创建的 Deployment。
  * 创建 Deployment `nfs-client-provisioner` 来添加连接到 NFS 服务器的信息，例如 `PROVISIONER_NAME` 是 `k8s-sigs.io/nfs-subdir-external-provisioner`。
  * 创建 StorageClass `nfs-client` 并链接到 `provisioner: k8s-sigs.io/nfs-subdir-external-provisioner`，相关的 PV 会自动创建。
  * 创建 PVC `nfs-pvc-from-sc` 并映射到 StorageClass `nfs-client` 上的 PV。
* 配置Configuration：
  * 创建一个 ConfigMap 以包含文件的内容，并将此 ConfigMap 挂载到 Pod 中的特定文件中。
  * 创建一个 ConfigMap 来包含用户名和密码，并在 Pod 中使用它们。
  * 在 Pod 中将 ConfigMap 用作环境变量。

建议：

* 首先删除 PVC，然后再删除 PV。
* 如果删除 PVC 时遇到 `Terminating` 状态，使用 `kubectl edit pvc <your_pvc_name>` 命令，然后删除 `finalize: <your_value>`。

## emptyDir

创建一个名为 `hello-producer` 的 Pod，并使用 `emptyDir` 类型的 Volume。

```bash
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

查看Pod `hello-producer`的状态。

```bash
kubectl get pod hello-producer -owide
```

Pod `hello-producer` 运行在节点node `cka003`上。

```console
NAME             READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
hello-producer   1/1     Running   0          6s    10.244.102.24   cka003   <none>           <none>
```

登录 `cka003`，因为 Pod `hello-producer` 正在该节点上运行。

为 `crictl` 命令设置环境变量 `CONTAINER_RUNTIME_ENDPOINT`。建议在所有节点上执行相同的操作。

```bash
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
```

运行命令 `crictl ps` 来获取 Pod `hello-producer` 的容器 ID。

```bash
crictl ps |grep hello-producer
```

容器 `producer` 的ID是 `05f5e1bb6a1bb`。

```console
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
50058afb3cba5       62aedd01bd852       About an hour ago   Running             producer            0                   e6953bd4833a7       hello-producer
```

运行命令 `crictl inspect`，获取已挂载的 `shared-volume` 的路径，它是 `emptyDir` 类型的。

```bash
crictl inspect 50058afb3cba5 | grep source | grep empty
```

运行结果

```console
"source": "/var/lib/kubelet/pods/d7424f86-534a-48f9-9001-9d2a6e822b12/volumes/kubernetes.io~empty-dir/shared-volume",
```

修改路径为上面获取到的 `shared-volume` 的挂载路径。然后我们会看到文件 `hello` 中的内容 `hello world`。

```bash
cd /var/lib/kubelet/pods/d7424f86-534a-48f9-9001-9d2a6e822b12/volumes/kubernetes.io~empty-dir/shared-volume
cat hello
```

Pod内的路径`/producer_dir`被挂载到了本地宿主机路径`/var/lib/kubelet/pods/d7424f86-534a-48f9-9001-9d2a6e822b12/volumes/kubernetes.io~empty-dir/shared-volume`。

我们在Pod内创建的文件`/producer_dir/hello`实际上在宿主机本地路径中。

让我们删除容器`producer`，容器`producer`将以新的容器ID重新启动，而文件`hello`仍将存在。

```bash
crictl ps
crictl stop <your_container_id>
crictl rm <your_container_id>
```

现在删除Pod `hello-producer`，容器`producer`会被删除，文件`hello`也会被删除。

```bash
kubectl delete pod hello-producer 
```

## hostPath

应用以下 yaml 文件创建一个 MySQL Pod 并挂载一个 `hostPath`。
将主机目录 `/tmp/mysql` 挂载到 Pod 目录 `/var/lib/mysql`。
在本地检查是否存在目录 `/tmp/mysql`，如果不存在，则执行命令 `mkdir /tmp/mysql` 创建它。

```bash
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

验证 MySQL 可用性。

检查 MySQL Pod 的状态。需要记录 Pod 的名称和其所运行的节点。

```bash
kubectl get pod -l app=mysql -o wide
```

运行结果

```console
NAME                     READY   STATUS              RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
mysql-749c8ddd67-h2rgs   0/1     ContainerCreating   0          28s   <none>   cka003   <none>           <none>
```

在MySQL Pod运行的节点登陆进入pod内部。

```bash
kubectl exec -it <your_pod_name> -- bash
```

在 Pod 中，进入 `/var/lib/mysql` 目录，该目录中的所有文件都与节点 `cka003` 上 `/tmp/mysql` 目录中的所有文件相同。

连接到 Pod 中的数据库。

```bash
mysql -h 127.0.0.1 -uroot -ppassword
```

执行下面命令对数据库进行简单的操作。

```bash
mysql> show databases;
mysql> connect mysql;
mysql> show tables;
mysql> exit
```

## PV和PVC

下面的演示中，我们将使用NFS作为后端存储来演示如何部署PV和PVC。

### 设置NFS共享

1. 安装nfs-kernel-server

登录到节点`cka002`。配置Worker `cka002`成为NFS服务器。

```bash
sudo apt-get install -y nfs-kernel-server
```

2.配置共享目录

创建共享文件夹。  

```bash
mkdir /nfsdata
```

编辑文件`/etc/exports`，添加一行`/nfsdata *(rw,sync,no_root_squash)`。

```bash
cat >> /etc/exports << EOF
/nfsdata *(rw,sync,no_root_squash)
EOF
```

有许多不同的NFS共享选项，例如：

* `*`：对所有IP或特定IP可访问。
* `rw`：作为读写共享。请注意，正常的Linux权限仍然适用。（请注意，这是默认选项。）
* `ro`：作为只读共享。
* `sync`：文件数据更改会立即写入磁盘，这会影响性能，但不太可能导致数据丢失。在某些发行版上，这是默认选项。
* `async`：与sync相反，文件数据更改最初写入内存。这提高了性能，但更容易导致数据丢失。在某些发行版上，这是默认选项。
* `root_squash`：将NFS客户端的root用户和组帐户映射到匿名帐户，通常是nobody帐户或nfsnobody帐户。有关更多详细信息，请参见本文后续的“用户ID映射”。（请注意，这是默认选项。）
* `no_root_squash`：将NFS客户端的root用户和组帐户映射到本地root和组帐户。

我们将使用基于Linux服务器之间的`nfs`和`rpcbind`服务的无密码远程挂载，而不是基于`smb`服务。首先，这两台服务器必须授权、安装并设置nfs和rpcbind服务，设置共享目录，启动服务，并在客户端上进行挂载。

启动`rpcbind`服务。

```bash
sudo systemctl enable rpcbind
sudo systemctl restart rpcbind
```

启动`nfs`服务。

```bash
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

如果`/etc/exports`文件被修改，我们需要运行下面的命令使之生效。

```bash
exportfs -ra
```

运行结果

```bash
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "*:/nfsdata".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x
```

检查共享目录是否已经被正确配置了。

```bash
showmount -e
```

如果看到下面的结果，则说明共享目录已经被正确配置了。

```console
Export list for cka002:
/nfsdata *
```

3.安装NFS客户端

在所有节点上安装NFS客户端。

```bash
sudo apt-get install -y nfs-common
```

4.验证NFS服务

登录到任何一个节点来验证NFS服务是否正确工作，以及NFS服务所共享到目录是否可见。
登陆到`cka001`，并检查`cka002`的共享目录状态。

```bash
showmount -e cka002
```

如果得到类似下面的结果，则说明NFS服务正常工作，包括共享目录。

```bash
Export list for cka002:
/nfsdata *
```

5.挂载NFS共享目录

执行下面命令，挂载NFS共享目录到任何一个非NFS服务器节点，比如`cka001` or `cka003`。

```bash
mkdir /remote-nfs-dir
mount -t nfs cka002:/nfsdata /remote-nfs-dir/
```

执行命令`df -h`来检查NFS挂载点是否正确，类似下面的结果。

```bash
Filesystem       Size  Used Avail Use% Mounted on
cka002:/nfsdata   40G  5.8G   32G  16% /remote-nfs-dir
```

### 创建 PV

创建一个 PV `mysql-pv`。
将 NFS 服务器 IP 替换为实际的 IP（这里是 `<cka002_ip>`），它是运行 NFS 服务器 `cka002` 的 IP。

```bash
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

执行下面的命令，检查创建的PV。

```bash
kubectl get pv
```

运行结果

```bash
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mysql-pv   1Gi        RWO            Retain           Available           nfs                     19s
```

### 创建 PVC

创建 PVC `mysql-pvc` 并指定存储大小、访问模式和存储类。
PVC `mysql-pvc` 将通过存储类名称自动与 PV 绑定。

```bash
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

### 消费 PVC

更新Deployment `mysql` 来使用之前创建的PVC `mysql-pvc`。

```bash
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

现在我们可以看到 MySQL 文件已经移动到了 `cka002` 的 `/nfsdata` 目录下。

## StorageClass

### 配置RBAC权限

RBAC（Role-Based Access Control，基于角色的访问控制）是 Kubernetes 中的一种授权机制，用于限制用户对资源的访问权限。
我们可以为 Kubernetes 集群中的用户分配不同的角色，以限制他们在集群中的操作。

要配置 RBAC 授权，需要执行以下步骤：

1. 为用户创建帐户
2. 为帐户创建角色
3. 为角色授予权限
4. 将角色绑定到帐户

这些步骤中的每一步都需要创建 Kubernetes 对象，例如 `ServiceAccount`、`Role`、`ClusterRole`、`RoleBinding` 和 `ClusterRoleBinding`。

RBAC权限使用`rbac.authorization.k8s.io` API组来驱动授权决策，允许我们通过Kubernetes API动态配置策略。

* ServiceAccount：`nfs-client-provisioner`
* 命名空间：`dev`
* ClusterRole：`nfs-client-provisioner-runner`。在节点、pv、pvc、sc和事件上授予授权（authorization）。
* ClusterRoleBinding：`run-nfs-client-provisioner`，将上述ClusterRole绑定到上述ServiceAccount。
* Role：`leader-locking-nfs-client-provisioner`。在endpoint上授予权限。
* RoleBinding：`leader-locking-nfs-client-provisioner`，将上述Role绑定到上述ServiceAccount。

创建RBAC权限。

```bash
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

### 创建Provisioner的Deloyment

"Provisioner" 可以翻译成 "提供程序"，这个词可以指为 Kubernetes 集群中提供各种资源的服务程序，如动态存储卷提供程序 (Dynamic Provisioner)、网络存储卷提供程序 (CSI Driver) 等。

创建 `nfs-client-provisioner` 部署，使用挂载到 `<cka002_ip>`(`cka002`) 上的 `/nfsdata` 目录的卷 `nfs-client-root`。
把 NFS 服务器的 IP 替换为实际的 IP 地址即可（这里用 `<cka002_ip>` 表示）。

```bash
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

### 创建 NFS StorageClass

创建 StorageClass `nfs-client`，定义 NFS 子目录外部 provisioner 的 Kubernetes Storage Class。

执行下面的命令编辑`nfs-storageclass.yaml`文件。

```bash
vi nfs-storageclass.yaml
```

添加下面的信息来配置 NFS StorageClass。

```bash
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

应用上面的yaml文件，使之生效。

```console
kubectl apply -f nfs-storageclass.yaml
```

### 创建PVC

创建 PVC `nfs-pvc-from-sc`。

```bash
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

查看所创建的 PVC `nfs-pvc-from-sc` 的状态。

```bash
kubectl get pvc nfs-pvc-from-sc
```

PVC `nfs-pvc-from-sc` 的当前状态是 `Pending`.

```console
NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-from-sc   Pending                                      nfs-client     112s
```

检查 `Pending` 状态的原因。

```bash
kubectl describe pvc nfs-pvc-from-sc
```

下面的信息说明 PVC `nfs-pvc-from-sc` 处于挂起状态，在等待卷（volume）创建完成。

```console
Events:
  Type    Reason                Age               From                         Message
  ----    ------                ----              ----                         -------
  Normal  ExternalProvisioning  9s (x6 over 84s)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "k8s-sigs.io/nfs-subdir-external-provisioner" or manually created by system administrator
```

### 消费PVC

创建 Deployment `mysql-with-sc-pvc` 以使用 PVC `nfs-pvc-from-sc`。

```bash
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

检查 Deployment `mysql-with-sc-pvc` 的状态。

```bash
kubectl get deployment mysql-with-sc-pvc -o wide
```

运行结果：

```console
NAME                READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES      SELECTOR
mysql-with-sc-pvc   1/1     1            1           16s   mysql        mysql:8.0   app=mysql
```

使用 Deployment `mysql-with-sc-pvc` 消费 PVC `nfs-pvc-from-sc` 后，PVC 的状态从 `Pending` 变为了 `Bound`。

```bash
kubectl get pvc nfs-pvc-from-sc
```

运行结果：

```console
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-from-sc   Bound    pvc-edf70dff-7407-4b38-aac9-9c2dd6a84316   1Gi        RWX            nfs-client     52m
```

检查相关 Pod 的状态。注意，Pod `mysql-with-sc-pvc-7c97d875f8-dwfkc` 运行在 `cka003` 上。

```bash
kubectl get pod -o wide -l app=mysql
```

运行结果：

```console
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
mysql-774db46945-h82kk               1/1     Running   0          69m     10.244.112.26   cka002   <none>           <none>
mysql-with-sc-pvc-7c97d875f8-wkvr9   1/1     Running   0          3m37s   10.244.102.27   cka003   <none>           <none>
```

我们现在来查看 NFS 服务器 `cka002` 上的共享目录 `/nfsdata/`。

```bash
ll /nfsdata/
```

NFS 服务器 `cka002` 上的共享目录 `/nfsdata/` 下有了2个子目录，与其他2个节点上的目录 `/remote-nfs-dir/` 下的内容是一致。

```bash
drwxrwxrwx  6 systemd-coredump root 4096 Jul 23 23:35 dev/
drwxr-xr-x  6 systemd-coredump root 4096 Jul 23 22:29 mysqldata/
```

命名空间Namespace的名称作为目录名在 `/nfsdata/` 目录下用于挂载到 Pod 中。
默认情况下，命名空间Namespace名称将用于挂载点。
如果我们想要使用自定义的文件夹来代替，我们需要声明一个 `nfs.io/storage-path` 注释，例如下面的例子。

在命名空间 `kube-system` 上创建 PVC `test-claim`，并消费 `nfs-client` 卷。

```bash
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

在上述情况下， PVC 创建在 `kube-system` 命名空间中，因此我们可以在节点 `cka002` 上的 `kube-system` 目录下看到 `test-path` 目录。

执行下面的命令，来查看目录 `/nfsdata/` 的整体目录结构。

```bash
tree -L 1 /nfsdata/ 
```

运行结果：

```console
/nfsdata/
├── dev
├── kube-system
└── mysqldata
```

注意：

上述规则遵循了 `nfs-subdir-external-provisioner` 实现，可能与其他`provisioner`不同。

参考：

 `nfs-subdir-external-provisioner` [项目](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)的详细信息。

## 配置Configuration

### ConfigMap

创建 ConfigMap `cm-nginx` 来配置文件 `nginx.conf`。

```bash
vi configmap.yaml
```

把下面的内容粘贴到文件`nginx.conf`中。

```yaml
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

应用文件`configmap.yaml`，创建ConfigMap。

```bash
kubectl apply -f configmap.yaml
```

创建Pod `nginx-with-cm`。

```bash
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

提示：

* 默认情况下，要挂载 ConfigMap，Kubernetes 会覆盖挂载点的所有内容。我们可以使用 `volumeMounts.subPath` 来指定只覆盖在 `mountPath` 中定义的 `nginx.conf` 文件。
* 如果我们使用 `volumeMounts.subPath` 来挂载一个容器卷，Kubernetes 将不会进行热更新以反映实时更新。

把从外部挂载的 `nginx.conf` 文件和上面的文件进行比较，以验证它是否已经被加载到容器中。

```bash
kubectl exec -it nginx-with-cm -- sh 
cat /etc/nginx/nginx.conf
```

### Secret

用base64方式编码密码。

```bash
echo -n admin | base64  
YWRtaW4=

echo -n 123456 | base64
MTIzNDU2
```

创建Secret `mysecret`。

```bash
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

使用卷将 Secret 挂载（注入，injection）到 Pod 中。

```bash
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

让我们登录进入到Pod `busybox-with-secret`内部，以验证 `mysecret` 的两个数据元素（`username`和`password`）是否已成功挂载到Pod中的路径 `/tmp/secret`。

```bash
kubectl exec -it busybox-with-secret -- sh
```

执行下面的命令，我们可以看到`mysecret` 的两个数据元素（`username`和`password`）已经以文件形式存在于目录`/tmp/secret`下。

```bash
/ # ls -l /tmp/secret/
lrwxrwxrwx    1 root     root            15 Jul 23 16:30 password -> ..data/password
lrwxrwxrwx    1 root     root            15 Jul 23 16:30 username -> ..data/username
```

而且我们可以看到这2个数据元素（`username`和`password`）的内容就是我们预先定义的。

```bash
/ # cat /tmp/secret/username
admin

/ # cat /tmp/secret/password
123456
```

### 拓展案例

#### 多种方法创建ConfigMap

我们可以通过文件、目录、或者值来创建ConfigMap。

下面我们创建ConfigMap `colors`，包含：

* 四个文件，文件名是四个颜色。
* 一个文件，文件名是最喜欢的颜色。

```bash
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

执行命令`tree configmap`，可以看到类似下面的文件目录结构。

```console
configmap
├── favorite
└── primary
    ├── black
    ├── cyan
    ├── magenta
    └── yellow
```

创建一个 ConfigMap，引用上面我们创建的文件。确保我们现在在路径 `~/configmap` 下。

```bash
kubectl create configmap colors \
--from-literal=text=black  \
--from-file=./favorite  \
--from-file=./primary/
```

查看ConfigMap `colors`的内容。

```bash
kubectl get configmap colors -o yaml
```

运行结果：

```bash
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

#### 通过ConfigMap设定环境变量

继续上面的例子，现在我们准备创建一个名为`pod-configmap-env`的Pod，设置环境变量`ilike`并从ConfigMap `colors`中分配值`favorite`。

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

连接并进入Pod `pod-configmap-env`内部。

```bash
kubectl exec -it pod-configmap-env -- bash
```

验证环境变量 `ilike` 的值是 `blue`，这是 ConfigMap `colors` 的 `favorite` 值。

```bash
root@pod-configmap-env:/# echo $ilike
blue
```

我们还可以使用 ConfigMap 的所有键值对来设置 Pod 的环境变量。

```bash
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

连接并进入Pod `pod-configmap-env-2`内部。

```bash
kubectl exec -it pod-configmap-env-2 -- bash
```

验证环境变量的值是我们在ConfigMap `colors`所定义的键值对。

```bash
root@pod-configmap-env-2:/# echo $black
k known as key
root@pod-configmap-env-2:/# echo $cyan
c
root@pod-configmap-env-2:/# echo $favorite
blue
```
