# CKA自学笔记19:Ingress-nginx

演示场景：

* 部署Ingress Controller。
* 创建两个Deployment `nginx-app-1`和`nginx-app-2`。
  * 在运行主机上创建主机目录 `/root/html-1` 和 `/root/html-2` 并挂载到两个Deployment上。
* 创建Service。
  * 创建Service `nginx-app-1`和`nginx-app-2`并将其映射到相关的Deployment`nginx-app-1`和`nginx-app-2`。
* 创建Ingress。
  * 创建Ingress资源`nginx-app`并将其映射到两个Services`nginx-app-1`和`nginx-app-1`。
* 测试可访问性。
  * 向Ingress中定义的两个主机发送HTTP请求。

参考：

* Github [ingress-nginx](https://github.com/kubernetes/ingress-nginx)
* [Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/)

## 部署Ingress控制器

获取Ingress控制器的yaml文件。最新版本的链接在[安装指南](https://kubernetes.github.io/ingress-nginx/deploy/)中。

```console
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```

修改`deploy.yaml`文件中镜像源为阿里云的源。

`deploy.yaml`文件中需要修改的行：

```yaml
image: k8s.gcr.io/ingress-nginx/controller:v1.2.1@sha256:5516d103a9c2ecc4f026efbd4b40662ce22dc1f824fb129ed121460aaa5c47f8
image: registry.k8s.io/ingress-nginx/controller:v1.3.0@sha256:d1707ca76d3b044ab8a28277a2466a02100ee9f58a86af1535a3edf9323ea1b5
image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
```

修改内容：

* `k8s.gcr.io/ingress-nginx/controller` 改为 `registry.aliyuncs.com/google_containers/nginx-ingress-controller`。
* `registry.k8s.io/ingress-nginx/controller` 改为 `registry.aliyuncs.com/google_containers/nginx-ingress-controller`。
* `k8s.gcr.io/ingress-nginx/kube-webhook-certgen` 改为 `registry.aliyuncs.com/google_containers/kube-webhook-certgen`。

修改命令：

```bash
sed -i 's/k8s.gcr.io\/ingress-nginx\/kube-webhook-certgen/registry.aliyuncs.com\/google\_containers\/kube-webhook-certgen/g' deploy.yaml
sed -i 's/k8s.gcr.io\/ingress-nginx\/controller/registry.aliyuncs.com\/google\_containers\/nginx-ingress-controller/g' deploy.yaml
```

应用文件 `deploy.yaml` 来创建 Ingress Nginx。

一个新的命名空间namespace `ingress-nginx` 会被创建，Ingress Nginx相关的资源运行在这个namespace上。

```bash
kubectl apply -f deploy.yaml
```

查看Pod的状态。

```bash
kubectl get pod -n ingress-nginx
```

确保所以pod的运行状态都正常，类似如下结果：

```console
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-lgtdj        0/1     Completed   0          49s
ingress-nginx-admission-patch-nk9fv         0/1     Completed   0          49s
ingress-nginx-controller-556fbd6d6f-6jl4x   1/1     Running     0          49s
```

## 本地测试方式

让我们创建一个简单的 Web 服务器和相关的服务：

```bash
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo
```

接下来创建一个 Ingress 资源。以下示例使用将主机映射到 localhost:

```bash
kubectl create ingress demo-localhost --class=nginx --rule="demo.localdev.me/*=demo:80"
```

现在，将本地端口转发到Ingress控制器：

```bash
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
```

现在，在另一个终端中访问 <http://demo.localdev.me:8080/>，我们应该看到一个 HTML 页面，上面写着 "It works!"。

```bash
curl http://demo.localdev.me:8080/
```

运行结果；

```html
<html><body><h1>It works!</h1></body></html>
```

删除演示中创建的临时资源。

```bash
kubectl delete ingress demo-localhost
kubectl delete service demo
kubectl delete deployment demo
```

## 创建Deployments

创建2个deployment `nginx-app-1` 和 `nginx-app-2`。

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-1
spec:
  selector:
    matchLabels:
      app: nginx-app-1
  replicas: 1 
  template:
    metadata:
      labels:
        app: nginx-app-1
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html
      volumes:
       - name: html
         hostPath:
           path: /opt/html-1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-2
spec:
  selector:
    matchLabels:
      app: nginx-app-2
  replicas: 1 
  template:
    metadata:
      labels:
        app: nginx-app-2
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html
      volumes:
       - name: html
         hostPath:
           path: /opt/html-2
EOF
```

执行命令 `kubectl get pod -o wide` 来获取pod的状态。
可以看到一个pod运行在节点 `cka002`，另外一个pod运行在节点`cka003`。

* Directory `/opt/html-2/` is on `cka002`
* Directory `/opt/html-1/` is on `cka002`

通过`curl`命令来访问这2个pod，收到`403 Forbidden`错误。

```bash
curl 10.244.102.13
curl 10.244.112.19
```

登录到节点`cka002`，执行下面命令，在`/opt/html-2/`路径下创建文件`index.html`。

```bash
cat <<EOF | sudo tee /opt/html-2/index.html
This is test 2 !!
EOF
```

登录到节点`cka003`，执行下面命令，在`/opt/html-1/`路径下创建文件`index.html`。

```bash
cat <<EOF | sudo tee /opt/html-1/index.html
This is test 1 !!
EOF
```

执行命令`kubectl get pod -o wide`，再次检查2个pod的状态。

```bash
curl 10.244.102.13
curl 10.244.112.19
```

现在可以通过`curl`命令来访问这2个pod了，收到了正确的回复信息。

```console
This is test 1 !!
This is test 2 !!
```

## 创建Service

创建 `nginx-app-1` 和 `nginx-app-2` 这2个 Service，并将它们分别映射到相关的 Deployment `nginx-app-1` 和 `nginx-app-2`。

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
 name: nginx-app-1
spec:
 ports:
 - protocol: TCP
   port: 80
   targetPort: 80
 selector:
   app: nginx-app-1
---
kind: Service
apiVersion: v1
metadata:
 name: nginx-app-2
spec:
 ports:
 - protocol: TCP
   port: 80
   targetPort: 80
 selector:
   app: nginx-app-2
EOF
```

检查刚刚创建的service的状态。

```bash
kubectl get svc -o wide
```

尝试通过`curl`命令来访问刚刚创建的service。

```bash
curl 11.244.165.64
curl 11.244.222.177
```

收到了正确的信息，说明访问成功。

```console
This is test 1 !!
This is test 2 !!
```

## 创建Ingress

创建 Ingress 资源 `nginx-app`，将其映射到我们创建的两个 Service `nginx-app-1` 和 `nginx-app-2`，并将其所在的 namespace 改为 `default`。

```bash
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-app
  namespace: default
spec:
  ingressClassName: "nginx"
  rules:
  - host: app1.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx-app-1
            port: 
              number: 80
  - host: app2.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx-app-2
            port:
              number: 80
EOF
```

查看所创建的Ingress资源的状态。

```bash
kubectl get ingress
```

## 可访问性测试

执行以下命令，来确定 Ingress 控制器正在节点 `cka003` 上运行。

```bash
kubectl get pod -n ingress-nginx -o wide
```

在当前节点上更新 `/etc/hosts` 文件。
将节点`cka003`的IP地址与两个主机名`app1.com`和`app1.com`进行映射，这些主机名表示Services`nginx-app-1`和`nginx-app-2`。
Ingress Controllers正在节点`cka003`上运行。

```bash
cat <<EOF | sudo tee -a /etc/hosts
<cka003_ip>  app1.com
<cka003_ip>  app2.com
EOF
```

执行以下命令，可以得到 IP 地址或 FQDN。

```bash
kubectl get service ingress-nginx-controller --namespace=ingress-nginx
```

在输出结果中可以看到 `EXTERNAL-IP` 字段。如果该字段像下面一样显示为 `<pending>`，这意味着 Kubernetes 集群无法提供负载均衡器（通常是因为它不支持 `LoadBalancer` 类型的服务）。

由于没有配置阿里云 ELB，因此有以下两个选项可以解决这个问题。

选项 1：手动将节点 IP 添加到运行 ingress 控制器的节点上。

执行命令 `kubectl get pod -n ingress-nginx -o wide` 来查看 ingress 控制器 pod 运行在哪个节点上。

手动将 `cka003` 的外部 IP 补丁到 `EXTERNAL-IP` 字段。

```bash
kubectl patch svc ingress-nginx-controller \
 --namespace=ingress-nginx \
 -p '{"spec": {"type": "LoadBalancer", "externalIPs":["<cka003_ip>"]}}'
```

选项 2：将 ingress 控制器从 `LoadBalancer` 类型更改为 `NodePort` 类型。

两个 Pod 中各有一个 `index.html` 文件，Web 服务通过节点 IP 对外暴露。`ingress-nginx-controller` 作为中心入口点，为来自 Pod 的不同后端服务提供了两个端口。

发送HTTP请求到在Ingress中定义的2个主机节点。

```bash
curl http://app1.com:30011
curl http://app2.com:30011

curl app1.com:30011
curl app2.com:30011
```

可以得到下面的信息，说明访问成功。

```console
This is test 1 !!
This is test 2 !!
```

删除上面演示中创建的临时资源。

```bash
kubectl delete ingress ingress-nginx-app
kubectl delete service nginx-app-1
kubectl delete service nginx-app-2
kubectl delete deployment nginx-app-1
kubectl delete deployment nginx-app-2
```
