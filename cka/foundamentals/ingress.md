# Ingress-nginx

!!! Scenario
    * Deploy Ingress Controller.
    * Create two deployment `nginx-app-1` and `nginx-app-2`.
        * Host directory `/root/html-1` and `/root/html-2` will be created and mounted to two Deployments on running host.
    * Create Service.
        * Create Service `nginx-app-1` and `nginx-app-2` and map to related Deployment `nginx-app-1` and `nginx-app-2`.
    * Create Ingress.
        * Create Ingress resource `nginx-app` and map to two Services `nginx-app-1` and `nginx-app-1`.
    * Test Accessibility.
        * Send HTTP request to two hosts defined in Ingress


!!! Reference
    * Github [ingress-nginx](https://github.com/kubernetes/ingress-nginx)
    * [Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/)



## Deploy Ingress Controller

Get Ingress Controller yaml file. The latest version link is in [Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/).
```console
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```

Below two images's sources needto be changed to Aliyun.
```
image: k8s.gcr.io/ingress-nginx/controller:v1.2.1@sha256:5516d103a9c2ecc4f026efbd4b40662ce22dc1f824fb129ed121460aaa5c47f8
image: registry.k8s.io/ingress-nginx/controller:v1.3.0@sha256:d1707ca76d3b044ab8a28277a2466a02100ee9f58a86af1535a3edf9323ea1b5
```
```

image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
```

From grc.io to Aliyun.

* `k8s.gcr.io/ingress-nginx/controller` to `registry.aliyuncs.com/google_containers/nginx-ingress-controller`
* `registry.k8s.io/ingress-nginx/controller` to `registry.aliyuncs.com/google_containers/nginx-ingress-controller`
* 
* `k8s.gcr.io/ingress-nginx/kube-webhook-certgen` to `registry.aliyuncs.com/google_containers/kube-webhook-certgen`


Commands:
```console
sed -i 's/k8s.gcr.io\/ingress-nginx\/kube-webhook-certgen/registry.aliyuncs.com\/google\_containers\/kube-webhook-certgen/g' deploy.yaml
sed -i 's/k8s.gcr.io\/ingress-nginx\/controller/registry.aliyuncs.com\/google\_containers\/nginx-ingress-controller/g' deploy.yaml
```

Apply the yaml file `deploy.yaml` to create Ingress Nginx.

A new namespace `ingress-nginx` was created and Ingress Nginx resources are running under the new namespace.
```console
kubectl apply -f deploy.yaml
```

Check the status of Pod.
```console
kubectl get pod -n ingress-nginx
```
Make sure all pods are not in error status, like below.
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-lgtdj        0/1     Completed   0          49s
ingress-nginx-admission-patch-nk9fv         0/1     Completed   0          49s
ingress-nginx-controller-556fbd6d6f-6jl4x   1/1     Running     0          49s
```

## Local testing

Let's create a simple web server and the associated service:
```console
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo
```

Then create an ingress resource. The following example uses a host that maps to localhost:
```console
kubectl create ingress demo-localhost --class=nginx --rule="demo.localdev.me/*=demo:80"
```

Now, forward a local port to the ingress controller:
```console
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
```

At this point, open another terminal to access http://demo.localdev.me:8080/, we should see an HTML page telling you "It works!".
```console
curl http://demo.localdev.me:8080/
```
Result
```
<html><body><h1>It works!</h1></body></html>
```

Clean up.
```
kubectl delete ingress demo-localhost
kubectl delete service demo
kubectl delete deployment demo
```





## Create Deployments

Create two deployment `nginx-app-1` and `nginx-app-2`.
```console
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

Get status of Pods by executing `kubectl get pod -o wide`. 
One pod is running on node `cka002`, another pod is running on node `cka003`.

* Directory `/opt/html-2/` is on `cka002`
* Directory `/opt/html-1/` is on `cka002`

Access to two Pod via curl. We get `403 Forbidden` error.
```console
curl 10.244.102.13
curl 10.244.112.19
```

Log onto node `cka002`, create `index.html` file in path `/opt/html-2/` with below command.
```console
cat <<EOF | sudo tee /opt/html-2/index.html
This is test 2 !!
EOF
```

Log onto node `cka003`, create `index.html` file in path `/opt/html-1/` with below command.
```console
cat <<EOF | sudo tee /opt/html-1/index.html
This is test 1 !!
EOF
```

Check Pods status again by executing `kubectl get pod -o wide`.

Now access to two Pod via `curl` is reachable. 
```console
curl 10.244.102.13
curl 10.244.112.19
```

We get correct information now.
```
This is test 1 !!
This is test 2 !!
```



## Create Service

Create Service `nginx-app-1` and `nginx-app-2` and map to related deployment `nginx-app-1` and `nginx-app-2`.
```console
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

Check the status by executing below comamnd.
```console
kubectl get svc -o wide
```

Access to two Service via `curl`. 
```console
curl 11.244.165.64
curl 11.244.222.177
```

We get correct information.
```
This is test 1 !!
This is test 2 !!
```





## Create Ingress

Create Ingress resource `nginx-app` and map to two Services `nginx-app-1` and `nginx-app-1` we created.
Change the namespace to `default`. 
```console
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

Get Ingress status by executing command below.
```console
kubectl get ingress
```



## Test Accessiblity

By executing command below, we know Ingress Controllers are running on node `cka003`.
```console
kubectl get pod -n ingress-nginx -o wide
```

Update `/etc/hosts` file in current node. 

Add mapping between node `cka003` IP and two host names `app1.com` and `app1.com` which present Services `nginx-app-1` and `nginx-app-2`. 

Ingress Controllers are running on node `cka003`

```console
cat <<EOF | sudo tee -a /etc/hosts
<cka003_ip>  app1.com
<cka003_ip>  app2.com
EOF
```


Get IP address or FQDN with the following command:
```console
kubectl get service ingress-nginx-controller --namespace=ingress-nginx
```

It will be the `EXTERNAL-IP` field. If that field shows `<pending>` like below, this means that the Kubernetes cluster wasn't able to provision the load balancer (generally, this is because it doesn't support services of type LoadBalancer).

As there is no Aliyun ELB configured, use below two options to make the external IP in place.

>  Option 1: manually add node ip to ingress controller, which the controller pod is running on.
>  
>  Execute command `kubectl get pod -n ingress-nginx -o wide` to see that ingress controller pod is running on node `cka003`.
>  
>  Manually patch the external ip of `cak003` to the `EXTERNAL-IP` field.
>  ```console
>  kubectl patch svc ingress-nginx-controller \
>    --namespace=ingress-nginx \
>    -p '{"spec": {"type": "LoadBalancer", "externalIPs":["<cka003_ip>"]}}'
>  ```
>  
>  Option 2: change ingress controller from `LoadBalancer` to `NodePort`.


Two files `index.html` are in two Pods, the web services are exposed to outside via node IP. 
The `ingress-nginx-controller` plays a central entry point for outside access, and provide two ports for different backend services from Pods.

Send HTTP request to two hosts defined in Ingress.
```console
curl http://app1.com:30011
curl http://app2.com:30011

curl app1.com:30011
curl app2.com:30011
```

Get below successful information.
```
This is test 1 !!
This is test 2 !!
```

Clean up.
```
kubectl delete ingress ingress-nginx-app
kubectl delete service nginx-app-1
kubectl delete service nginx-app-2
kubectl delete deployment nginx-app-1
kubectl delete deployment nginx-app-2
```







