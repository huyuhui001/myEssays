# Network Policy

## Replace Flannel by Calico

!!! Scenario
    * Remove Flannel
    * Install Calico


Demo:

If Calico was installed at the installation phase, ignore this section.

Delete Flannel
```console
kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/v0.18.1/Documentation/kube-flannel.yml
```
or
```console
kubectl delete -f kube-flannel.yml
```
Output:
```
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy "psp.flannel.unprivileged" deleted
clusterrole.rbac.authorization.k8s.io "flannel" deleted
clusterrolebinding.rbac.authorization.k8s.io "flannel" deleted
serviceaccount "flannel" deleted
configmap "kube-flannel-cfg" deleted
daemonset.apps "kube-flannel-ds" deleted
```


Clean up iptables for all nodes.
```console
rm -rf /var/run/flannel /opt/cni /etc/cni /var/lib/cni
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

Log out and log on to host (e.g., cka001) again. Install Calico.
```console
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```
Output:
```
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

Verify status of Calico. Make sure all Pods are running
```console
kubectl get pod -n kube-system | grep calico
```
Output:
```
NAME                                       READY   STATUS        RESTARTS   AGE
calico-kube-controllers-7bc6547ffb-tjfcg   1/1     Running       0          30m
calico-node-7x8jm                          1/1     Running       0          30m
calico-node-cwxj5                          1/1     Running       0          30m
calico-node-rq978                          1/1     Running       0          30m
```

If facing any error, check log in the Container.
```console
# Get Container ID
crictl ps

# Get log info
crictl logs <your_container_id>
```


As we change CNI from Flannel to Calico, we need delete all Pods. All of Pods will be created automatically again. 
```console
kubectl delete pod -A --all
```

Make sure all Pods are up and running successfully.
```console
kubectl get pod -A
```







## Inbound Rules

!!! Scenario
    * Create workload for test.
    * Deny For All Ingress
    * Allow For Specific Ingress
    * Verify NetworkPolicy

Demo:

### Create workload for test.

Create three Deployments `pod-netpol-1`,`pod-netpol-2`,`pod-netpol-3` based on image `busybox`.
```console
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

Check Pods IP.
```console
kubectl get pod -owide
```
Output:
```
NAME                                      READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
pod-netpol-1-6494f6bf8b-n58r9             1/1     Running   0          29s   10.244.102.30   cka003   <none>           <none>
pod-netpol-2-77478d77ff-l6rzm             1/1     Running   0          29s   10.244.112.30   cka002   <none>           <none>
pod-netpol-3-68977dcb48-ql5s6             1/1     Running   0          29s   10.244.102.31   cka003   <none>           <none>
```

Attach to Pod `pod-netpol-1`
```console
kubectl exec -it pod-netpol-1-6494f6bf8b-n58r9 -- sh
```

Execute command `ping` that `pod-netpol-2` and `pod-netpol-3` are both reachable. 
```console
/ # ping 10.244.112.30 
3 packets transmitted, 3 packets received, 0% packet loss

/ # ping 10.244.102.31
3 packets transmitted, 3 packets received, 0% packet loss
```



### Deny For All Ingress

Create deny policy for all ingress.
```console
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

Attach to Pod `pod-netpol-1` again
```console
kubectl exec -it pod-netpol-1-6494f6bf8b-n58r9 -- sh
```

Execute command `ping` that `pod-netpol-2` and `pod-netpol-3` are both unreachable as expected.
```console
/ # ping 10.244.112.30
3 packets transmitted, 0 packets received, 100% packet loss

/ # ping 10.244.102.31
3 packets transmitted, 0 packets received, 100% packet loss
```



### Allow For Specific Ingress

Create NetworkPlicy to allow ingress from `pod-netpol-1` to `pod-netpol-2`.
```console
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

### Verify NetworkPolicy

Attach to Pod `pod-netpol-1` again
```console
kubectl exec -it pod-netpol-1-6494f6bf8b-n58r9 -- sh
```

Execute command `ping` to check if `pod-netpol-2` and `pod-netpol-3` are reachable. 
As expected, `pod-netpol-2` is reachable and `pod-netpol-3` is still unreachable. 
```console
/ # ping 10.244.112.30
3 packets transmitted, 3 packets received, 0% packet loss

/ # ping 10.244.102.8
3 packets transmitted, 0 packets received, 100% packet loss
```



## Inbound Across Namespace

!!! Scenario
   * Create workload and namespace for test
   * Create Allow Ingress
   * Verify Policy

Demo:

### Create workload and namespace for test

Create Namespace `ns-netpol`.
```console
kubectl create ns ns-netpol
```

Create Deployment `pod-netpol`.
```console
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

Check Pod status on new namespace.
```console
kubectl get pod -n ns-netpol
```
Output:
```
NAME                          READY   STATUS    RESTARTS   AGE
pod-netpol-5b67b6b496-2cgnw   1/1     Running   0          9s
```

Attach into `pod-netpol` Pod.
```console
kubectl exec -it pod-netpol-5b67b6b496-2cgnw -n ns-netpol -- sh
```

Try to ping pod-netpol-2 (`10.244.112.30`) in Namespace `dev`. It's unreachable. 
```console
ping 10.244.112.30
3 packets transmitted, 0 packets received, 100% packet loss
```



### Create Allow Ingress

Create NetworkPolicy to allow access to pod-netpol-2 in namespace `dev` from all Pods in namespace `pod-netpol`.
```console
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



### Verify Policy

Attach into `pod-netpol` Pod.
```console
kubectl exec -it pod-netpol-5b67b6b496-2cgnw -n ns-netpol -- sh
```

Try to ping pod-netpol-2 (`10.244.112.30`) in Namespace `dev`. It's still unreachable. 
```console
ping 10.244.112.30
3 packets transmitted, 0 packets received, 100% packet loss
```

What we allowed ingress is from namespace with label `allow: to-pod-netpol-2`, but namespace `ns-netpol` does not have it and we need label it.
```console
kubectl label ns ns-netpol allow=to-pod-netpol-2
```

Attach into `pod-netpol` Pod.
```console
kubectl exec -it pod-netpol-5b67b6b496-2cgnw -n ns-netpol -- sh
```

Try to ping pod-netpol-2 (`10.244.112.30`) in Namespace `dev`. It's now reachable. 
```console
ping 10.244.112.30
3 packets transmitted, 3 packets received, 0% packet loss
```

Be noted that we can use namespace default label as well.





## NetworkPolicy

!!! Scenario
    Ingress

    * Create two namespaces `my-ns-1`, `my-ns-2`.
    * Create two deployments on `my-ns-1`, `nginx` listens to port `80` and `tomcat` listens to port `8080`.
    * Create NetworkPolicy `my-networkpolicy-1` on namespace `my-ns-1` to allow access to port 8080 from namespace `my-ns-1`.
    * Verify the access to `nginx` port `80` and `tomcat` port `8080`.
    * Edit the NetworkPolicy to allow access to port 8080 from namespace `my-ns-2`.
    * Verify the access to `nginx` port `80` and `tomcat` port `8080`.

Demo:

Create namespaces
```console
kubectl create namespace my-ns-1
kubectl create namespace my-ns-2
```

Create deployment on `my-ns-1`
```console
kubectl create deployment my-nginx --image=nginx --namespace=my-ns-1 --port=80
kubectl create deployment my-tomcat --image=tomcat --namespace=my-ns-1 --port=8080
```

Get the label: e.g., `kubernetes.io/metadata.name=my-ns-1`, `kubernetes.io/metadata.name=my-ns-2`.
```console
kubectl get namespace my-ns-1 --show-labels  
kubectl get namespace my-ns-2 --show-labels   
```

Create NetworkPolicy to allow access from my-ns-2 to Pod with port 8080 on my-ns-1.
Refer to yaml template from the link https://kubernetes.io/docs/concepts/services-networking/network-policies/.
```console
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

Check Deployment and Pod status
```console
kubectl get deployment,pod -n my-ns-1 -o wide
```

Create temp pod on namespace `my-ns-1`.
Attach to the pod and verify the access. 
Command `curl <nginx_ip>:80` failed. 
Command `curl <tomcat_ip>:80` succeed. 
```console
kubectl run centos --image=centos -n my-ns-1 -- "/bin/sh" "-c" "sleep 3600"
kubectl exec -it mycentos -n my-ns-1 -- bash
```

Create temp pod on namespace `my-ns-2`.
Attach to the pod and verify the access. 
Command `curl <nginx_ip>:80` failed. 
Command `curl <tomcat_ip>:80` failed. 
```console
kubectl run centos --image=centos -n my-ns-2 -- "/bin/sh" "-c" "sleep 3600"
kubectl exec -it mycentos -n my-ns-2 -- bash
```

Edit `my-networkpolicy-1` to change `ingress.from.namespaceSelector.matchLabels` to `my-ns-2`.

Attach to temp pod on namespace `my-ns-2`.
Verify the access. 
Command `curl <nginx_ip>:80` failed. 
Command `curl <tomcat_ip>:80` succeed. 
```console
kubectl exec -it mycentos -n my-ns-2 -- bash
```


Clean up:
```console
kubectl delete namespace my-ns-1
kubectl delete namespace my-ns-2
```

