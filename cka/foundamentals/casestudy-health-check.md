
# Case Study: Health Check

!!! Scenario
    * Create Deployment and Service
    * Simulate an error (delete index.html)
    * Pod is in unhealth status and is removed from endpoint list
    * Fix the error (revert the index.html)
    * Pod is back to normal and in endpoint list


## Create Deployment and Service

Create Deployment `nginx-healthcheck` and Service `nginx-healthcheck`.
```console
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

Check Pod `nginx-healthcheck`.
```console
kubectl get pod -owide
```
Result
```
NAME                                 READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
nginx-healthcheck-79fc55d944-jw887   1/1     Running   0          9s    10.244.102.14   cka003   <none>           <none>
nginx-healthcheck-79fc55d944-nwwjc   1/1     Running   0          9s    10.244.112.13   cka002   <none>           <none>
```

Access Pod IP via `curl` command, e.g., above example.
```console
curl 10.244.102.14
curl 10.244.112.13
```
We see a successful `index.html` content of Nginx below with above example.

Check details of Service craeted in above example.
```console
kubectl describe svc nginx-healthcheck
```
We will see below output. There are two Pods information listed in `Endpoints`.
```
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

We can also get information of Endpoints.
```console
kubectl get endpoints nginx-healthcheck
```
Result
```
NAME                ENDPOINTS                           AGE
nginx-healthcheck   10.244.102.14:80,10.244.112.13:80   72s
```

Till now, two `nginx-healthcheck` Pods are working and providing service as expected. 


## Simulate readinessProbe Failure

Let's simulate an error by deleting and `index.html` file in on of `nginx-healthcheck` Pod and see what's readinessProbe will do.

First, execute `kubectl exec -it <your_pod_name> -- bash` to log into `nginx-healthcheck` Pod, and delete the `index.html` file.
```console
kubectl exec -it nginx-healthcheck-79fc55d944-jw887 -- bash
cd /usr/share/nginx/html/
rm -rf index.html
exit
```

After that, let's check the status of above Pod that `index.html` file was deleted.
```console
kubectl describe pod nginx-healthcheck-79fc55d944-jw887
```
We can now see `Readiness probe failed` error event message.
```
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

Let's check another Pod. 
```console
kubectl describe pod nginx-healthcheck-79fc55d944-nwwjc
```
There is no error info.
```
......
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m46s  default-scheduler  Successfully assigned dev/nginx-healthcheck-79fc55d944-nwwjc to cka002
  Normal  Pulled     3m45s  kubelet            Container image "nginx:latest" already present on machine
  Normal  Created    3m45s  kubelet            Created container nginx-healthcheck
  Normal  Started    3m45s  kubelet            Started container nginx-healthcheck
```

Now, access Pod IP via `curl` command and see what the result of each Pod.
```console
curl 10.244.102.14
curl 10.244.112.13
```

Result: 

* `curl 10.244.102.14` failed with `403 Forbidden` error below. 
* `curl 10.244.112.13` works well.


Let's check current status of Nginx Service after one of Pods runs into failure. 
```console
kubectl describe svc nginx-healthcheck
```

In below output, there is only one Pod information listed in Endpoint.
```
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

Same result we can get by checking information of Endpoints, which is only Pod is running.
```console
kubectl get endpoints nginx-healthcheck 
```
Output:
```
NAME                ENDPOINTS          AGE
nginx-healthcheck   10.244.112.13:80   6m5s
```


## Fix readinessProbe Failure

Let's re-create the `index.html` file again in the Pod. 
```console
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

We now can see that two Pods are back to Endpoints to provide service now.
```console
kubectl describe svc nginx-healthcheck
kubectl get endpoints nginx-healthcheck
```

Re-access Pod IP via `curl` command and we can see both are back to normal status.
```console
curl 10.244.102.14
curl 10.244.112.13
```

Verify the Pod status again. 
```console
kubectl describe pod nginx-healthcheck-79fc55d944-jw887
```

!!! Conclusion

    By delete the `index.html` file, the Pod is in unhealth status and is removed from endpoint list. 

    One one health Pod can provide normal service.


Clean up
```console
kubectl delete service nginx-healthcheck
kubectl delete deployment nginx-healthcheck
```


## Simulate livenessProbe Failure

Re-create Deployment `nginx-healthcheck` and Service `nginx-healthcheck`.

Deployment:
```
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nginx-healthcheck        0/2     2            0           7s
```

Pods:
```
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-healthcheck-79fc55d944-lknp9        1/1     Running   0          96s
nginx-healthcheck-79fc55d944-wntmg        1/1     Running   0          96s
```

Change nginx default listening port from `80` to `90` to simulate livenessProbe Failure. livenessProbe check the live status via port `80`. 
```console
kubectl exec -it nginx-healthcheck-79fc55d944-lknp9 -- bash
root@nginx-healthcheck-79fc55d944-lknp9:/# cd /etc/nginx/conf.d
root@nginx-healthcheck-79fc55d944-lknp9:/etc/nginx/conf.d# sed -i 's/80/90/g' default.conf
root@nginx-healthcheck-79fc55d944-lknp9:/etc/nginx/conf.d# nginx -s reload
2022/07/24 12:59:45 [notice] 79#79: signal process started
```

The Pod runs into failure.
```console
kubectl describe pod nginx-healthcheck-79fc55d944-lknp9
```
We can see `livenessProbe` failed error event message.
```
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

Once failure detected by `livenessProbe`, the container will restarted again automatically. 
The `default.conf` we modified will be replaced by default file and the container status is up and normal.

