
# 主题讨论:健康检查

演示场景：

* 创建 Deployment 和 Service
* 模拟一个错误（删除 index.html）
* Pod 处于不健康状态并从 endpoint 列表中删除
* 修复错误（恢复 index.html）
* Pod 回到正常状态并重新加入 endpoint 列表

## 创建 Deployment 和 Service

创建Deployment `nginx-healthcheck` 和Service `nginx-healthcheck`。

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-healthcheck
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx-healthcheck
  template:
    metadata:
      labels:
        name: nginx-healthcheck
    spec:
      containers:
        - name: nginx-healthcheck
          image: nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80  
          livenessProbe:
            initialDelaySeconds: 5
            periodSeconds: 5
            tcpSocket:
              port: 80
            timeoutSeconds: 5   
          readinessProbe:
            httpGet:
              path: /
              port: 80
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-healthcheck
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    name: nginx-healthcheck
EOF

```

查询Pod `nginx-healthcheck`。

``` bash
kubectl get pod -owide
```

运行结果：

```console
NAME                                 READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
nginx-healthcheck-79fc55d944-jw887   1/1     Running   0          9s    10.244.102.14   cka003   <none>           <none>
nginx-healthcheck-79fc55d944-nwwjc   1/1     Running   0          9s    10.244.112.13   cka002   <none>           <none>
```

通过命令`curl`来访问上面运行结果中pod的IP地址。

```bash
curl 10.244.102.14
curl 10.244.112.13
```

如果上面命令成功执行，则会返回Nginx中`index.html`的内容。

获取前面创建的Service的详细信息。

```bash
kubectl describe svc nginx-healthcheck
```

输出结果如下。在 `Endpoints` 部分我们可以看到2个pod。

```yaml
Name:                     nginx-healthcheck
Namespace:                dev
Labels:                   <none>
Annotations:              <none>
Selector:                 name=nginx-healthcheck
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       11.244.238.20
IPs:                      11.244.238.20
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31795/TCP
Endpoints:                10.244.102.14:80,10.244.112.13:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

获取Endpoints的信息。

```bash
kubectl get endpoints nginx-healthcheck
```

运行结果

```console
NAME                ENDPOINTS                           AGE
nginx-healthcheck   10.244.102.14:80,10.244.112.13:80   72s
```

至此，2个pod `nginx-healthcheck` 都能按照我们的期望正常工作。

## 模拟readinessProbe错误

让我们通过删除 `nginx-healthcheck` Pod 中的 `index.html` 文件来模拟错误，观察 readinessProbe 的表现。

首先，执行 `kubectl exec -it <your_pod_name> -- bash` 命令以登录到 `nginx-healthcheck` Pod，并删除 `index.html` 文件。

```bash
kubectl exec -it nginx-healthcheck-79fc55d944-jw887 -- bash
cd /usr/share/nginx/html/
rm -rf index.html
exit
```

在执行了删除 `nginx-healthcheck` Pod 中的 `index.html` 文件之后，我们检查该 Pod 的状态。

```bash
kubectl describe pod nginx-healthcheck-79fc55d944-jw887
```

下面的输出结果中，我们可以看到 `Readiness probe failed` 这个错误事件信息。

```console
......
Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Normal   Scheduled  2m8s             default-scheduler  Successfully assigned dev/nginx-healthcheck-79fc55d944-jw887 to cka003
  Normal   Pulled     2m7s             kubelet            Container image "nginx:latest" already present on machine
  Normal   Created    2m7s             kubelet            Created container nginx-healthcheck
  Normal   Started    2m7s             kubelet            Started container nginx-healthcheck
  Warning  Unhealthy  2s (x2 over 7s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 403
```

检查另一个pod。

```bash
kubectl describe pod nginx-healthcheck-79fc55d944-nwwjc
```

下面的输出结果中，没有发现错误。

```console
......
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m46s  default-scheduler  Successfully assigned dev/nginx-healthcheck-79fc55d944-nwwjc to cka002
  Normal  Pulled     3m45s  kubelet            Container image "nginx:latest" already present on machine
  Normal  Created    3m45s  kubelet            Created container nginx-healthcheck
  Normal  Started    3m45s  kubelet            Started container nginx-healthcheck
```

现在，通过`curl`命令来访问2个pod的IP地址，我们来观察会得到怎样的结果。

```bash
curl 10.244.102.14
curl 10.244.112.13
```

运行结果：

* `curl 10.244.102.14` 失败，错误信息是 `403 Forbidden`。
* `curl 10.244.112.13` 成功。

我们现在来查询Nginx service在一个pod失败时的状态。

```bash
kubectl describe svc nginx-healthcheck
```

在下面的输出结果中，我们看到Endpoint部分中只有一个pod的信息。

```yaml
Name:                     nginx-healthcheck
Namespace:                dev
Labels:                   <none>
Annotations:              <none>
Selector:                 name=nginx-healthcheck
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       11.244.238.20
IPs:                      11.244.238.20
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31795/TCP
Endpoints:                10.244.112.13:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

同样的，我们可以通过检查Endpoint的信息，也能发现只有一个pod正在运行。

```bash
kubectl get endpoints nginx-healthcheck 
```

运行结果：

```console
NAME                ENDPOINTS          AGE
nginx-healthcheck   10.244.112.13:80   6m5s
```

## 修复readinessProbe错误

现在，我们在pod中重新创建 `index.html` 文件，来修复错误。

```bash
kubectl exec -it nginx-healthcheck-79fc55d944-jw887 -- bash

cd /usr/share/nginx/html/

cat > index.html << EOF 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
EOF

exit
```

现在我们可以看到两个Pod已经重新加入了Endpoint列表，可以提供服务了。

```bash
kubectl describe svc nginx-healthcheck
kubectl get endpoints nginx-healthcheck
```

重新通过`curl`命令访问2个pod的IP地址，我们可以看到它们都已经恢复到正常状态了。

```bash
curl 10.244.102.14
curl 10.244.112.13
```

再次验证pod的状态。

```bash
kubectl describe pod nginx-healthcheck-79fc55d944-jw887
```

结论：

* 通过删除 `index.html` 文件，Pod 进入不健康状态并从端点列表中删除。
* 只有一个健康的 Pod 可以提供正常的服务。

清除演示中创建的临时资源。

```bash
kubectl delete service nginx-healthcheck
kubectl delete deployment nginx-healthcheck
```

## 模拟livenessProbe错误

重新创建deployment `nginx-healthcheck` 和service `nginx-healthcheck`。

Deployment:

```console
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nginx-healthcheck        0/2     2            0           7s
```

Pods:

```console
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-healthcheck-79fc55d944-lknp9        1/1     Running   0          96s
nginx-healthcheck-79fc55d944-wntmg        1/1     Running   0          96s
```

将 Nginx 默认监听端口从 `80` 改为 `90`，以模拟 livenessProbe 失败。livenessProbe 通过端口 `80` 检查生存状态。

```bash
kubectl exec -it nginx-healthcheck-79fc55d944-lknp9 -- bash
root@nginx-healthcheck-79fc55d944-lknp9:/# cd /etc/nginx/conf.d
root@nginx-healthcheck-79fc55d944-lknp9:/etc/nginx/conf.d# sed -i 's/80/90/g' default.conf
root@nginx-healthcheck-79fc55d944-lknp9:/etc/nginx/conf.d# nginx -s reload
2022/07/24 12:59:45 [notice] 79#79: signal process started
```

Pod现在表现为失败状态。

```bash
kubectl describe pod nginx-healthcheck-79fc55d944-lknp9
```

在pod的事件信息中，我们可以发现 `livenessProbe` 错误信息。

```console
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  17m                    default-scheduler  Successfully assigned dev/nginx-healthcheck-79fc55d944-lknp9 to cka003
  Normal   Pulled     2m47s (x2 over 17m)    kubelet            Container image "nginx:latest" already present on machine
  Normal   Created    2m47s (x2 over 17m)    kubelet            Created container nginx-healthcheck
  Normal   Started    2m47s (x2 over 17m)    kubelet            Started container nginx-healthcheck
  Warning  Unhealthy  2m47s (x4 over 2m57s)  kubelet            Readiness probe failed: Get "http://10.244.102.46:80/": dial tcp 10.244.102.46:80: connect: connection refused
  Warning  Unhealthy  2m47s (x3 over 2m57s)  kubelet            Liveness probe failed: dial tcp 10.244.102.46:80: connect: connection refused
  Normal   Killing    2m47s                  kubelet            Container nginx-healthcheck failed liveness probe, will be restarted
```

当`livenessProbe`检测到失败后，容器将自动重新启动。我们修改的`default.conf`文件将被默认文件替换，容器状态将恢复正常。
