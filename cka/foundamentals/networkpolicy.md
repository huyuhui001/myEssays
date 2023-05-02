# CKA自学笔记23:Network Policy

## 用Calico替换Flannel

演示场景：

* 卸载Flannel
* 安装Calico

演示：

如果在安装过程中已经安装了 Calico，则可以忽略这部分内容。

卸载Flannel

```bash
kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/v0.18.1/Documentation/kube-flannel.yml
```

或者

```bash
kubectl delete -f kube-flannel.yml
```

输出：

```console
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy "psp.flannel.unprivileged" deleted
clusterrole.rbac.authorization.k8s.io "flannel" deleted
clusterrolebinding.rbac.authorization.k8s.io "flannel" deleted
serviceaccount "flannel" deleted
configmap "kube-flannel-cfg" deleted
daemonset.apps "kube-flannel-ds" deleted
```

在所有节点上清除iptables设置。

```bash
rm -rf /var/run/flannel /opt/cni /etc/cni /var/lib/cni
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

重新登录主机节点，例如 `cka001`，安装Calico，

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Output:

```console
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created
```

验证Calico安装状态，确保在每个节点上都正常运行。

```bash
kubectl get pod -n kube-system | grep calico
```

输出结果：

```console
NAME                                       READY   STATUS        RESTARTS   AGE
calico-kube-controllers-7bc6547ffb-tjfcg   1/1     Running       0          30m
calico-node-7x8jm                          1/1     Running       0          30m
calico-node-cwxj5                          1/1     Running       0          30m
calico-node-rq978                          1/1     Running       0          30m
```

如果遇到任何错误，首先检查容器container日志。

```bash
# Get Container ID
crictl ps

# Get log info
crictl logs <your_container_id>
```

由于我们将 CNI 从 Flannel 更改为 Calico，我们需要删除所有 Pod，所有 Pod 都将自动重新创建。

```bash
kubectl delete pod -A --all
```

查询所有pod都状态，确保他们都正常运行。

```bash
kubectl get pod -A
```

## 入站规则（Inbound Rules）

演示场景：

* 创建用于测试的工作负载。
* 禁止所有入站流量。
* 允许特定的入站流量。
* 验证NetworkPolicy。

### 创建测试工作负载

创建三个 Deployment，名称为 `pod-netpol-1`、`pod-netpol-2` 和 `pod-netpol-3`，它们都基于镜像 `busybox`。

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pod-netpol-1
  name: pod-netpol-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-netpol-1
  template:
    metadata:
      labels:
        app: pod-netpol-1
    spec:
      containers:
      - image: busybox
        name: busybox
        command: ["sh", "-c", "sleep 1h"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pod-netpol-2
  name: pod-netpol-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-netpol-2
  template:
    metadata:
      labels:
        app: pod-netpol-2
    spec:
      containers:
      - image: busybox
        name: busybox
        command: ["sh", "-c", "sleep 1h"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pod-netpol-3
  name: pod-netpol-3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-netpol-3
  template:
    metadata:
      labels:
        app: pod-netpol-3
    spec:
      containers:
      - image: busybox
        name: busybox
        command: ["sh", "-c", "sleep 1h"]       
EOF
```

检查pod的IP地址：

```bash
kubectl get pod -owide
```

输出结果：

```console
NAME                                      READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
pod-netpol-1-6494f6bf8b-n58r9             1/1     Running   0          29s   10.244.102.30   cka003   <none>           <none>
pod-netpol-2-77478d77ff-l6rzm             1/1     Running   0          29s   10.244.112.30   cka002   <none>           <none>
pod-netpol-3-68977dcb48-ql5s6             1/1     Running   0          29s   10.244.102.31   cka003   <none>           <none>
```

登录进入pod `pod-netpol-1`。

```bash
kubectl exec -it pod-netpol-1-6494f6bf8b-n58r9 -- sh
```

执行命令 `ping`，确保 `pod-netpol-2` 和 `pod-netpol-3` 可互相访问。

```bash
/ # ping 10.244.112.30 
3 packets transmitted, 3 packets received, 0% packet loss

/ # ping 10.244.102.31
3 packets transmitted, 3 packets received, 0% packet loss
```

### 禁止所有入站流量

创建策略，禁止所有入站流量。

```bash
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

再次登录进入pod `pod-netpol-1` 。

```bash
kubectl exec -it pod-netpol-1-6494f6bf8b-n58r9 -- sh
```

Execute command `ping` that `pod-netpol-2` and `pod-netpol-3` are both unreachable as expected.
执行命令 `ping`，和我们预期一样，`pod-netpol-2` 和 `pod-netpol-3` 此时互相无法访问。

```bash
/ # ping 10.244.112.30
3 packets transmitted, 0 packets received, 100% packet loss

/ # ping 10.244.102.31
3 packets transmitted, 0 packets received, 100% packet loss
```

### 允许特定的入站流量

创建 NetworkPolicy，允许来自 `pod-netpol-1` 到 `pod-netpol-2` 的入站流量。

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-pod-netpol-1-to-pod-netpol-2
spec:
  podSelector:
    matchLabels:
      app: pod-netpol-2
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: pod-netpol-1
EOF
```

### 验证NetworkPolicy

再次登录进入pod `pod-netpol-1`。

```bash
kubectl exec -it pod-netpol-1-6494f6bf8b-n58r9 -- sh
```

和我们设定的预期一致，`pod-netpol-2`可以访问，但是`pod-netpol-3`仍然无法访问。

```bash
/ # ping 10.244.112.30
3 packets transmitted, 3 packets received, 0% packet loss

/ # ping 10.244.102.8
3 packets transmitted, 0 packets received, 100% packet loss
```

## 跨namespace的入站流量

演示场景：

* 创建工作负载和测试的namespace
* 创建允许 Ingress 的 NetworkPolicy
* 验证 NetworkPolicy

### 创建测试工作负载和namespace

创建namespace `ns-netpol`。

```bash
kubectl create ns ns-netpol
```

创建 deployment `pod-netpol`。

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pod-netpol
  name: pod-netpol
  namespace: ns-netpol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-netpol
  template:
    metadata:
      labels:
        app: pod-netpol
    spec:
      containers:
      - image: busybox
        name: busybox
        command: ["sh", "-c", "sleep 1h"]
EOF
```

在新的namespace上检查pod的运行状态。

```bash
kubectl get pod -n ns-netpol
```

输出结果：

```console
NAME                          READY   STATUS    RESTARTS   AGE
pod-netpol-5b67b6b496-2cgnw   1/1     Running   0          9s
```

连接登入pod `pod-netpol`。

```bash
kubectl exec -it pod-netpol-5b67b6b496-2cgnw -n ns-netpol -- sh
```

在namespace `dev` 中 ping `pod-netpol-2`（`10.244.112.30`）。

```bash
ping 10.244.112.30
```

运行结果，pod无法访问。

```console
3 packets transmitted, 0 packets received, 100% packet loss
```

### 创建允许入站流量的Ingress

创建 `NetworkPolicy`，允许来自namespace `pod-netpol` 中的所有 Pod 访问namespace `dev` 中的 `pod-netpol-2`。

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ns-netpol-to-pod-netpol-2
spec:
  podSelector:
    matchLabels:
      app: pod-netpol-2
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          allow: to-pod-netpol-2
EOF
```

### 验证策略

登录进入pod `pod-netpol`。

```bash
kubectl exec -it pod-netpol-5b67b6b496-2cgnw -n ns-netpol -- sh
```

尝试在namespace `dev` 中对 `pod-netpol-2` (`10.244.112.30`) 执行 ping 命令。

```bash
ping 10.244.112.30
```

运行结果如下，依然无法访问。

```console
3 packets transmitted, 0 packets received, 100% packet loss
```

我们允许的入站流量是来自带有标签 `allow: to-pod-netpol-2` 的命名空间，但命名空间 `ns-netpol` 没有这个标签，我们需要给它打上标签。

```bash
kubectl label ns ns-netpol allow=to-pod-netpol-2
```

登录进入pod `pod-netpol`。

```bash
kubectl exec -it pod-netpol-5b67b6b496-2cgnw -n ns-netpol -- sh
```

尝试在namespace `dev` 中对 `pod-netpol-2` (`10.244.112.30`) 执行 ping 命令。

```bash
ping 10.244.112.30
```

运行结果如下，可以访问。

```console
3 packets transmitted, 3 packets received, 0% packet loss
```

注意，我们也可以使用命名空间的默认标签。

## NetworkPolicy

Ingress演示场景：

* 创建两个namespace `my-ns-1` 和 `my-ns-2`。
* 在 `my-ns-1` 中创建两个部署，`nginx` 监听端口 `80`，`tomcat` 监听端口 `8080`。
* 在namespace `my-ns-1` 中创建 NetworkPolicy `my-networkpolicy-1`，以允许从namespace `my-ns-1` 访问端口 `8080`。
* 验证对 `nginx` 端口 `80` 和 `tomcat` 端口 `8080` 的访问。
* 编辑 NetworkPolicy，以允许从命名空间 `my-ns-2` 访问端口 `8080`。
* 验证对 `nginx` 端口 `80` 和 `tomcat` 端口 `8080` 的访问。

演示：

创建2个namespaces。

```bash
kubectl create namespace my-ns-1
kubectl create namespace my-ns-2
```

在namespace `my-ns-1`上创建deployment。

```bash
kubectl create deployment my-nginx --image=nginx --namespace=my-ns-1 --port=80
kubectl create deployment my-tomcat --image=tomcat --namespace=my-ns-1 --port=8080
```

查询2个namespace的标签，例如：`kubernetes.io/metadata.name=my-ns-1`, `kubernetes.io/metadata.name=my-ns-2`.

```bash
kubectl get namespace my-ns-1 --show-labels  
kubectl get namespace my-ns-2 --show-labels   
```

创建 NetworkPolicy，允许从 `my-ns-2` 访问 `my-ns-1` 上监听 8080 端口的 Pod。

参考：[yaml文件模版](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

```bash
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-networkpolicy-1
  namespace: my-ns-1
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: my-ns-1
      ports:
        - protocol: TCP
          port: 8080
EOF
```

查询 deployment 和 pod 的状态。

```bash
kubectl get deployment,pod -n my-ns-1 -o wide
```

在namespace `my-ns-1` 中创建一个临时的 Pod。

登录进入到这个 Pod，执行下面2个命令，验证访问。

* 运行 `curl <nginx_ip>:80` 失败
* 运行 `curl <tomcat_ip>:80` 成功

```bash
kubectl run centos --image=centos -n my-ns-1 -- "/bin/sh" "-c" "sleep 3600"
kubectl exec -it mycentos -n my-ns-1 -- bash
```

在namespace `my-ns-2` 中创建一个临时 Pod，然后连接到该 Pod 并验证访问。

* 命令 `curl <nginx_ip>:80` 失败
* 命令 `curl <tomcat_ip>:80` 失败。

```bash
kubectl run centos --image=centos -n my-ns-2 -- "/bin/sh" "-c" "sleep 3600"
kubectl exec -it mycentos -n my-ns-2 -- bash
```

修改 `my-networkpolicy-1`， 把 `ingress.from.namespaceSelector.matchLabels` 的值改为 `my-ns-2`。

登录进入namespace `my-ns-2`上的临时pod，验证访问。

* 命令 `curl <nginx_ip>:80` 失败
* 命令 `curl <tomcat_ip>:80` 成功

```bash
kubectl exec -it mycentos -n my-ns-2 -- bash
```

删除演示中创建的临时资源。

```bash
kubectl delete namespace my-ns-1
kubectl delete namespace my-ns-2
```
