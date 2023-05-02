# CKA自学笔记24:Cluster Management

演示场景：`etcd` Backup and Restore

* 安装 `etcdctl`
* 在备份之前创建 Deployment
* 备份 `etcd`
* 在备份之后创建 Deployment
* 停止服务
* 停止 etcd
* 恢复 `etcd`
* 启动服务
* 验证

## `etcd`备份和恢复

### 安装`etcdctl`

下载`etcd`安装包。

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.5.3/etcd-v3.5.3-linux-amd64.tar.gz
```

解压`etcd`安装包，并赋予执行权限。

```bash
tar -zxvf etcd-v3.5.3-linux-amd64.tar.gz
cp etcd-v3.5.3-linux-amd64/etcdctl /usr/local/bin/
sudo chmod u+x /usr/local/bin/etcdctl
```

验证：

```bash
etcdctl --help
```

### 创建一个deployment（备份前）

备份前创建一个deployment。

```bash
kubectl create deployment app-before-backup --image=nginx
```

### 备份`etcd`

说明：

* `<CONTROL_PLANE_IP_ADDRESS>` 是控制平面节点的实际IP地址。
* `--endpoints`：指定 etcd 备份的保存位置，2379 是 etcd 的端口号。
* `--cert`：指定 etcd 证书的位置，证书是由 `kubeadm` 生成并保存在 `/etc/kubernetes/pki/etcd/` 目录下的。
* `--key`：指定 etcd 证书的私钥的位置，证书是由 `kubeadm` 生成并保存在 `/etc/kubernetes/pki/etcd/` 目录下的。
* `--cacert`：指定 etcd 证书的 CA 的位置，证书是由 `kubeadm` 生成并保存在 `/etc/kubernetes/pki/etcd/` 目录下的。

```bash
etcdctl \
  --endpoints=https://<CONTROL_PLANE_IP_ADDRESS>:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  snapshot save snapshot-$(date +"%Y%m%d%H%M%S").db
```

或者

```bash
etcdctl \
  --endpoints=https://<cka001_ip>:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  snapshot save snapshot-$(date +"%Y%m%d%H%M%S").db
```

Output:

```console
{"level":"info","ts":"2022-07-24T18:51:21.328+0800","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file","path":"snapshot-20220724185121.db.part"}
{"level":"info","ts":"2022-07-24T18:51:21.337+0800","logger":"client","caller":"v3/maintenance.go:211","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"2022-07-24T18:51:21.337+0800","caller":"snapshot/v3_snapshot.go:73","msg":"fetching snapshot","endpoint":"https://<cka001_ip>:2379"}
{"level":"info","ts":"2022-07-24T18:51:21.415+0800","logger":"client","caller":"v3/maintenance.go:219","msg":"completed snapshot read; closing"}
{"level":"info","ts":"2022-07-24T18:51:21.477+0800","caller":"snapshot/v3_snapshot.go:88","msg":"fetched snapshot","endpoint":"https://<cka001_ip>:2379","size":"3.6 MB","took":"now"}
{"level":"info","ts":"2022-07-24T18:51:21.477+0800","caller":"snapshot/v3_snapshot.go:97","msg":"saved","path":"snapshot-20220724185121.db"}
Snapshot saved at snapshot-20220724185121.db
```

执行命令`ls -al`在当前目录中读取我们刚刚创建的备份文件。

```bash
-rw-------  1 root root 3616800 Jul 24 18:51 snapshot-20220724185121.db
```

### 创建一个deployment（备份后）

备份后，我们创建另外一个deployment。

```bash
kubectl create deployment app-after-backup --image=nginx
```

删除备份前我们创建的那个deployment。

```bash
kubectl delete deployment app-before-backup
```

检查deployment的状态。

```bash
kubectl get deploy
```

运行结果：

```console
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
app-after-backup         1/1     1            1           108s
```

### 停止Services

删除`etcd`的目录。

```bash
mv /var/lib/etcd/ /var/lib/etcd.bak
```

停止 `kubelet`。

```bash
systemctl stop kubelet
```

停止 `kube-apiserver`。

```bash
nerdctl -n k8s.io ps -a | grep apiserver
```

运行结果：

```console
0c5e69118f1b    registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0             "kube-apiserver --ad…"    32 hours ago    Up                  k8s://kube-system/kube-apiserver-cka001/kube-apiserver
638bb602c310    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  32 hours ago    Up                  k8s://kube-system/kube-apiserver-cka001
```

停止那些仍旧处于 `up` 状态的容器。

```bash
nerdctl -n k8s.io stop <container_id>

nerdctl -n k8s.io stop 0c5e69118f1b
nerdctl -n k8s.io stop 638bb602c310
```

直至没有处于`up`状态的`kube-apiserver`。

```bash
nerdctl -n k8s.io ps -a | grep apiserver
```

运行结果：

```console
0c5e69118f1b    registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0             "kube-apiserver --ad…"    32 hours ago    Created             k8s://kube-system/kube-apiserver-cka001/kube-apiserver
638bb602c310    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  32 hours ago    Created             k8s://kube-system/kube-apiserver-cka001
```

### 停止etcd

```bash
nerdctl -n k8s.io ps -a | grep etcd
```

运行结果：

```console
0965b195f41a    registry.aliyuncs.com/google_containers/etcd:3.5.1-0                       "etcd --advertise-cl…"    32 hours ago    Up                  k8s://kube-system/etcd-cka001/etcd
9e1bea9f25d1    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  32 hours ago    Up                  k8s://kube-system/etcd-cka001
```

停止那些仍旧处于 `up` 状态的容器。

```bash
nerdctl -n k8s.io stop <container_id>
```

```console
nerdctl -n k8s.io stop 0965b195f41a
nerdctl -n k8s.io stop 9e1bea9f25d1
```

直至没有处于`up`状态的`kube-apiserver`。

```bash
nerdctl -n k8s.io ps -a | grep etcd
```

运行结果：

```console
0965b195f41a    registry.aliyuncs.com/google_containers/etcd:3.5.1-0                       "etcd --advertise-cl…"    32 hours ago    Created             k8s://kube-system/etcd-cka001/etcd
9e1bea9f25d1    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  32 hours ago    Created             k8s://kube-system/etcd-cka001
```

### 恢复`etcd`

在控制平面节点上执行恢复操作，使用实际的备份文件，这里是文件 `snapshot-20220724185121.db`。

```bash
etcdctl snapshot restore snapshot-20220724185121.db \
    --endpoints=<cka001_ip>:2379 \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt\
    --data-dir=/var/lib/etcd
```

运行结果：

```console
Deprecated: Use `etcdutl snapshot restore` instead.

2022-07-24T18:57:49+08:00       info    snapshot/v3_snapshot.go:248     restoring snapshot      {"path": "snapshot-20220724185121.db", "wal-dir": "/var/lib/etcd/member/wal", "data-dir": "/var/lib/etcd", "snap-dir": "/var/lib/etcd/member/snap", "stack": "go.etcd.io/etcd/etcdutl/v3/snapshot.(*v3Manager).Restore\n\t/go/src/go.etcd.io/etcd/release/etcd/etcdutl/snapshot/v3_snapshot.go:254\ngo.etcd.io/etcd/etcdutl/v3/etcdutl.SnapshotRestoreCommandFunc\n\t/go/src/go.etcd.io/etcd/release/etcd/etcdutl/etcdutl/snapshot_command.go:147\ngo.etcd.io/etcd/etcdctl/v3/ctlv3/command.snapshotRestoreCommandFunc\n\t/go/src/go.etcd.io/etcd/release/etcd/etcdctl/ctlv3/command/snapshot_command.go:129\ngithub.com/spf13/cobra.(*Command).execute\n\t/go/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:856\ngithub.com/spf13/cobra.(*Command).ExecuteC\n\t/go/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:960\ngithub.com/spf13/cobra.(*Command).Execute\n\t/go/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:897\ngo.etcd.io/etcd/etcdctl/v3/ctlv3.Start\n\t/go/src/go.etcd.io/etcd/release/etcd/etcdctl/ctlv3/ctl.go:107\ngo.etcd.io/etcd/etcdctl/v3/ctlv3.MustStart\n\t/go/src/go.etcd.io/etcd/release/etcd/etcdctl/ctlv3/ctl.go:111\nmain.main\n\t/go/src/go.etcd.io/etcd/release/etcd/etcdctl/main.go:59\nruntime.main\n\t/go/gos/go1.16.15/src/runtime/proc.go:225"}
2022-07-24T18:57:49+08:00       info    membership/store.go:141 Trimming membership information from the backend...
2022-07-24T18:57:49+08:00       info    membership/cluster.go:421       added member    {"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["http://localhost:2380"]}
2022-07-24T18:57:49+08:00       info    snapshot/v3_snapshot.go:269     restored snapshot       {"path": "snapshot-20220724185121.db", "wal-dir": "/var/lib/etcd/member/wal", "data-dir": "/var/lib/etcd", "snap-dir": "/var/lib/etcd/member/snap"}
```

检查被删除的`etcd`目录是否已经从备份中恢复了。

```bash
tree /var/lib/etcd
```

运行结果：

```console
/var/lib/etcd
└── member
    ├── snap
    │   ├── 0000000000000001-0000000000000001.snap
    │   └── db
    └── wal
        └── 0000000000000000-0000000000000000.wal
```

### 启动服务Services

启动 `kubelet`。服务 `kube-apiserver` 和 `etcd` 也会继 `kubelet` 启动后被自动启动。

```bash
systemctl start kubelet
```

执行下面命令确认所有服务都已经启动和正常运行。

```bash
systemctl status kubelet.service
nerdctl -n k8s.io ps -a | grep etcd
nerdctl -n k8s.io ps -a | grep apiserver
```

查看当前 `etcd` 的状态。

```console
0965b195f41a    registry.aliyuncs.com/google_containers/etcd:3.5.1-0                       "etcd --advertise-cl…"    32 hours ago     Created             k8s://kube-system/etcd-cka001/etcd
3b8f37c87782    registry.aliyuncs.com/google_containers/etcd:3.5.1-0                       "etcd --advertise-cl…"    6 seconds ago    Up                  k8s://kube-system/etcd-cka001/etcd
9e1bea9f25d1    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  32 hours ago     Created             k8s://kube-system/etcd-cka001
fbbbb628a945    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  6 seconds ago    Up                  k8s://kube-system/etcd-cka001
```

查看当前 `apiserver` 的状态。

```console
0c5e69118f1b    registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0             "kube-apiserver --ad…"    32 hours ago      Created             k8s://kube-system/kube-apiserver-cka001/kube-apiserver
281cf4c6670d    registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0             "kube-apiserver --ad…"    14 seconds ago    Up                  k8s://kube-system/kube-apiserver-cka001/kube-apiserver
5ed8295d92da    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  15 seconds ago    Up                  k8s://kube-system/kube-apiserver-cka001
638bb602c310    registry.aliyuncs.com/google_containers/pause:3.6                          "/pause"                  32 hours ago      Created             k8s://kube-system/kube-apiserver-cka001
```

### 验证

检查集群的状态，查看是否pod `app-before-backup` 存在。

```bash
kubectl get deploy
```

运行结果：

```console
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
app-before-backup        1/1     1            1           11m
```

## 集群升级

演示场景：集群升级

* 驱逐控制平面节点
* 检查当前可用的 `kubeadm` 版本
* 将 `kubeadm` 升级到新版本
* 检查升级计划
* 应用升级计划以升级到新版本
* 升级 `kubelet` 和 `kubectl`
* 启用控制平面节点调度
* 驱逐工作节点
* 升级 `kubeadm` 和 `kubelet`
* 启用工作节点调度

参考：[kubeadm升级](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

### 升级控制平面

#### 控制平面准备

驱逐控制平面节点。

```bash
kubectl drain <control_plane_node_name> --ignore-daemonsets 
```

这里是：

```bash
kubectl drain cka001 --ignore-daemonsets 
```

运行结果：

```console
node/cka001 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-dsx76, kube-system/kube-proxy-cm4hc
evicting pod kube-system/calico-kube-controllers-5c64b68895-jr4nl
evicting pod kube-system/coredns-6d8c4cb4d-g4jxc
evicting pod kube-system/coredns-6d8c4cb4d-sqcvj
pod/calico-kube-controllers-5c64b68895-jr4nl evicted
pod/coredns-6d8c4cb4d-g4jxc evicted
pod/coredns-6d8c4cb4d-sqcvj evicted
node/cka001 drained
```

控制平面节点现在处于 `SchedulingDisabled` 状态。

```bash
kubectl get node -owide
```

运行结果：

```console
NAME     STATUS                     ROLES                  AGE   VERSION   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cka001   Ready,SchedulingDisabled   control-plane,master   32h   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
cka002   Ready                      <none>                 32h   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
cka003   Ready                      <none>                 32h   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
```

查询当前 `kubeadm` 可用版本。

```bash
apt policy kubeadm
```

输出结果：

```console
kubeadm:
  Installed: 1.24.0-00
  Candidate: 1.24.3-00
  Version table:
     1.24.3-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
     1.24.2-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
     1.24.1-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
     1.24.0-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
     1.24.2-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
 *** 1.24.0-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
        100 /var/lib/dpkg/status
     1.23.7-00 500
        500 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
......
```

升级 `kubeadm` 到 `Candidate: 1.24.2-00` 版本。

```bash
sudo apt-get -y install kubeadm=1.24.2-00 --allow-downgrades
```

查询升级计划。

```bash
kubeadm upgrade plan
```

输出类似下面的升级计划。

```console
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.24.0
[upgrade/versions] kubeadm version: v1.24.2
I0724 19:05:00.111855 1142460 version.go:255] remote version is much newer: v1.24.3; falling back to: stable-1.23
[upgrade/versions] Target version: v1.24.2
[upgrade/versions] Latest version in the v1.23 series: v1.24.2

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     3 x v1.24.0   v1.24.2

Upgrade to the latest version in the v1.23 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.24.0   v1.24.2
kube-controller-manager   v1.24.0   v1.24.2
kube-scheduler            v1.24.0   v1.24.2
kube-proxy                v1.24.0   v1.24.2
CoreDNS                   v1.8.6    v1.8.6
etcd                      3.5.1-0   3.5.1-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.24.2

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

#### 控制平面升级

参考前面的升级计划，我们升级到 v1.24.2 版本。

```bash
kubeadm upgrade apply v1.24.2
```

通过选项 `--etcd-upgrade=false`，我们把`etcd`排除出当前升级范围。

```bash
kubeadm upgrade apply v1.24.2 --etcd-upgrade=false
```

收到下面的信息，则代表上面的升级命令成功了。

```console
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.24.2". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

升级 `kubelet` 和 `kubectl`。

```bash
sudo apt-get -y install kubelet=1.24.2-00 kubectl=1.24.2-00 --allow-downgrades
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

查询节点当前状态。

```bash
kubectl get node
```

运行结果：

```console
NAME     STATUS                     ROLES                  AGE   VERSION
cka001   Ready,SchedulingDisabled   control-plane,master   32h   v1.24.2
cka002   Ready                      <none>                 32h   v1.24.0
cka003   Ready                      <none>                 32h   v1.24.0
```

After verify that each node is in Ready status, enable node scheduling.

在确认所有节点都处于Ready状态，则激活scheduling。

```bash
kubectl uncordon <control_plane_node_name>
```

这里是：

```bash
kubectl uncordon cka001
```

输出结果：

```console
node/cka001 uncordoned
```

再次检查节点状态，确保所有节点都处于Ready状态。

```bash
kubectl get node
```

运行结果：

```console
NAME     STATUS   ROLES                  AGE   VERSION
cka001   Ready    control-plane,master   32h   v1.24.2
cka002   Ready    <none>                 32h   v1.24.0
cka003   Ready    <none>                 32h   v1.24.0
```

### 升级工作节点

#### 工作节点准备

登录节点 `cka001`。
驱逐 Worker 节点，需要显式指定是否删除本地存储。

```bash
kubectl drain <worker_node_name> --ignore-daemonsets --force
kubectl drain <worker_node_name> --ignore-daemonsets --delete-emptydir-data --force
```

如果遇到关于 `emptydir`依赖的错误，则执行第二个命令。

```bash
kubectl drain cka002 --ignore-daemonsets --force
kubectl drain cka002 --ignore-daemonsets --delete-emptydir-data --force
```

输出结果：

```console
node/cka002 cordoned
WARNING: deleting Pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: dev/ubuntu; ignoring DaemonSet-managed Pods: kube-system/calico-node-p5rf2, kube-system/kube-proxy-zvs68
evicting pod ns-netpol/pod-netpol-5b67b6b496-2cgnw
evicting pod dev/ubuntu
evicting pod dev/app-before-backup-66dc9d5cb-6xc8c
evicting pod dev/nfs-client-provisioner-86d7fb78b6-2f5dx
evicting pod dev/pod-netpol-2-77478d77ff-l6rzm
evicting pod ingress-nginx/ingress-nginx-admission-patch-nk9fv
evicting pod ingress-nginx/ingress-nginx-admission-create-lgtdj
evicting pod kube-system/coredns-6d8c4cb4d-l4kx4
pod/ingress-nginx-admission-create-lgtdj evicted
pod/ingress-nginx-admission-patch-nk9fv evicted
pod/nfs-client-provisioner-86d7fb78b6-2f5dx evicted
pod/app-before-backup-66dc9d5cb-6xc8c evicted
pod/coredns-6d8c4cb4d-l4kx4 evicted
pod/pod-netpol-5b67b6b496-2cgnw evicted
pod/pod-netpol-2-77478d77ff-l6rzm evicted
pod/ubuntu evicted
node/cka002 drained
```

#### 工作节点升级

登录节点 `cka002`。

下载 `kubeadm` 的 `v1.24.2`版本。

```bash
sudo apt-get -y install kubeadm=1.24.2-00 --allow-downgrades
```

升级 `kubeadm`。

```bash
sudo kubeadm upgrade node
```

升级 `kubelet`。

```bash
sudo apt-get -y install kubelet=1.24.2-00 --allow-downgrades
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

确认所有节点都处于Ready状态，则激活scheduling。

```bash
kubectl uncordon <worker_node_name>
```

这里是：

```bash
kubectl uncordon cka002
```

#### 工作节点验证

查询节点状态。

```bash
kubectl get node
```

运行结果：

```console
NAME     STATUS   ROLES                  AGE   VERSION
cka001   Ready    control-plane,master   32h   v1.24.2
cka002   Ready    <none>                 32h   v1.24.2
cka003   Ready    <none>                 32h   v1.24.0
```

在节点 `cka003` 上重复上面的步骤。

登录节点 `cka003`。如果遇到关于 `emptydir`依赖的错误，则执行第二个命令。

```bash
kubectl drain cka003 --ignore-daemonsets --ignore-daemonsets --force
kubectl drain cka003 --ignore-daemonsets --ignore-daemonsets --delete-emptydir-data --force
```

登录节点 `cka003`，执行下面的命令。

```bash
sudo apt-get -y install kubeadm=1.24.2-00 --allow-downgrades

sudo kubeadm upgrade node

sudo apt-get -y install kubelet=1.24.2-00 --allow-downgrades
sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl get node
kubectl uncordon cka003
```

查询节点状态。

```bash
kubectl get node
```

运行结果：

```console
NAME     STATUS   ROLES                  AGE   VERSION
cka001   Ready    control-plane,master   32h   v1.24.2
cka002   Ready    <none>                 32h   v1.24.2
cka003   Ready    <none>                 32h   v1.24.2
```

### 总结

在控制面板上的执行步骤：

```bash
kubectl get node -owide
kubectl drain cka001 --ignore-daemonsets
kubectl get node -owide
apt policy kubeadm
apt-get -y install kubeadm=1.24.0-00 --allow-downgrades
kubeadm upgrade plan
kubeadm upgrade apply v1.24.0
# kubeadm upgrade apply v1.24.0 --etcd-upgrade=false
apt-get -y install kubelet=1.24.0-00 kubectl=1.24.0-00 --allow-downgrades
systemctl daemon-reload
systemctl restart kubelet
kubectl get node
kubectl uncordon cka001
```

在工作节点上的执行步骤：

* 在控制面板上：

```bash
kubectl drain cka002 --ignore-daemonsets
```

* 在工作节点上：

```bash
apt-get -y install kubeadm=1.24.1-00 --allow-downgrades
kubeadm upgrade node
apt-get -y install kubelet=1.24.1-00 --allow-downgrades
systemctl daemon-reload
systemctl restart kubelet
kubectl uncordon cka002
```

在其他工作节点上重复上面的步骤。
