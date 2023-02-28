# Role Based Access Control (RBAC)

!!! Scenario
    1. Create differnet profiles for one cluster.
    2. Use `cfssl` generate certificates for each profile.
    3. Create new kubeconfig file with all profiles and associated users.
    4. Merge old and new kubeconfig files into new kubeconfig file. We can switch different context for further demo.

!!! Background
    * Role-based access control (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within the organization.
    * When using client certificate authentication, we can generate certificates manually through `easyrsa`, `openssl` or `cfssl`.
    
!!! Best-Pracice
    * The purpose of kubeconfig is to grant different authorizations to different users for different clusters. 
    * Different contexts will link to different clusters.
    * It's not recommended to put multiple users' contexts for one cluster in one kubeconfig. 
    * It's recommended to use one kubeconfig file for one user.



### Install cfssl

Install `cfssl` tool
```console
apt install golang-cfssl
```

### Set Multiple Contexts

#### Current Context

Execute command `kubectl config` to get current contenxt.
```console
kubectl config get-contexts
```
We get below key information of the cluster.

* Cluster Name: kubernetes
* System account: kubenetes-admin
* Current context name: kubernetes-admin@kubernetes (format: `<system_account>@<cluster_name>`)

```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```


#### Create CA Config File


Get overview of directory `/etc/kubernetes/pki`.
```console
tree /etc/kubernetes/pki
```
Result
```
/etc/kubernetes/pki
├── apiserver.crt
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── etcd
│   ├── ca.crt
│   ├── ca.key
│   ├── healthcheck-client.crt
│   ├── healthcheck-client.key
│   ├── peer.crt
│   ├── peer.key
│   ├── server.crt
│   └── server.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
└── sa.pub
```


Change to directory `/etc/kubernetes/pki`.
```console
cd /etc/kubernetes/pki
```

Check if file `ca-config.json` is in place in current directory.
```console
ll ca-config.json
```

If not, create it.

* We can add multiple profiles to specify different expiry date, scenario, parameters, etc.. 
* Profile will be used to sign certificate.
* `87600` hours are about 10 years.


Here we will create 1 additional profile `dev`.
```console
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "dev": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```



#### Create CSR file for signature

A CertificateSigningRequest (CSR) resource is used to request that a certificate be signed by a denoted signer, after which the request may be approved or denied before finally being signed.

It is important to set `CN` and `O` attribute of the CSR. 

* The `CN` is the *name of the user* to request CSR.
* The `O` is the *group* that this user will belong to. We can refer to RBAC for standard groups.

Stay in the directory `/etc/kubernetes/pki`.

Create csr file `cka-dev-csr.json`. 
`CN` is `cka-dev`.
`O` is `k8s`.

```console
cat > cka-dev-csr.json <<EOF
{
  "CN": "cka-dev",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

Generate certifcate and key for the profile we defined above.
`cfssljson -bare cka-dev` will generate two files, `cka-dev.pem` as public key and `cka-dev-key.pem` as private key.

```console
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=ca-config.json -profile=dev cka-dev-csr.json | cfssljson -bare cka-dev
```

Get below files.
```console
ll -tr | grep cka-dev
```
```
-rw-r--r-- 1 root root  222 Jul 24 08:49 cka-dev-csr.json
-rw-r--r-- 1 root root 1281 Jul 24 09:14 cka-dev.pem
-rw------- 1 root root 1675 Jul 24 09:14 cka-dev-key.pem
-rw-r--r-- 1 root root 1001 Jul 24 09:14 cka-dev.csr
```







#### Create file kubeconfig

Get the IP of Control Plane (e.g., `<cka001_ip>` here) to composite evn variable `APISERVER` (`https://<control_plane_ip>:<port>`).
```console
kubectl get node -owide
```
```
NAME     STATUS   ROLES                  AGE   VERSION  OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cka001   Ready    control-plane,master   14h   v1.24.0  Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
cka002   Ready    <none>                 14h   v1.24.0  Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
cka003   Ready    <none>                 14h   v1.24.0  Ubuntu 20.04.4 LTS   5.4.0-122-generic   containerd://1.5.9
```

Export env `APISERVER`.
```console
echo "export APISERVER=\"https://<cka001_ip>:6443\"" >> ~/.bashrc
source ~/.bashrc
```

Verify the setting.
```console
echo $APISERVER
```
Output:
```
https://<cka001_ip>:6443
```


1. Set up cluster

Stay in the directory `/etc/kubernetes/pki`.

Generate kubeconfig file.
```console
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${APISERVER} \
  --kubeconfig=cka-dev.kubeconfig
```

Now we get the new config file `cka-dev.kubeconfig`
```console
ll -tr | grep cka-dev
```
Output:
```
-rw-r--r-- 1 root root  222 Jul 24 08:49 cka-dev-csr.json
-rw-r--r-- 1 root root 1281 Jul 24 09:14 cka-dev.pem
-rw------- 1 root root 1675 Jul 24 09:14 cka-dev-key.pem
-rw-r--r-- 1 root root 1001 Jul 24 09:14 cka-dev.csr
-rw------- 1 root root 1671 Jul 24 09:16 cka-dev.kubeconfig
```

Get content of file `cka-dev.kubeconfig`.
```console
cat cka-dev.kubeconfig
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <your_key>
    server: https://<cka001_ip>:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```




2. Set up user

In file `cka-dev.kubeconfig`, user info is null. 

Set up user `cka-dev`.
```console
kubectl config set-credentials cka-dev \
  --client-certificate=/etc/kubernetes/pki/cka-dev.pem \
  --client-key=/etc/kubernetes/pki/cka-dev-key.pem \
  --embed-certs=true \
  --kubeconfig=cka-dev.kubeconfig
```

Now file `cka-dev.kubeconfig` was updated and user information was added.
```console
cat cka-dev.kubeconfig
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <your_key>
    server: https://<cka001_ip>:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: cka-dev
  user:
    client-certificate-data: <your_key>
    client-key-data: <your_key>
```

Now we have a complete kubeconfig file `cka-dev.kubeconfig`.
When we use it to get node information, receive error below because we did not set up current-context in kubeconfig file.
```console
kubectl --kubeconfig=cka-dev.kubeconfig get nodes
```
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Current contents is empty.
```console
kubectl --kubeconfig=cka-dev.kubeconfig config get-contexts
```
```
CURRENT   NAME   CLUSTER   AUTHINFO   NAMESPACE
```



3. Set up Context

Set up context. 
```console
kubectl config set-context dev --cluster=kubernetes --user=cka-dev  --kubeconfig=cka-dev.kubeconfig
```

Now we have context now but the `CURRENT` flag is empty.
```console
kubectl --kubeconfig=cka-dev.kubeconfig config get-contexts
```
Output:
```
CURRENT   NAME   CLUSTER      AUTHINFO   NAMESPACE
          dev    kubernetes   cka-dev 
```

Set up default context. The context will link clusters and users for multiple clusters environment and we can switch to different cluster.
```console
kubectl --kubeconfig=cka-dev.kubeconfig config use-context dev
```


4. Verify

Now `CURRENT` is marked with `*`, that is, current-context is set up.
```console
kubectl --kubeconfig=cka-dev.kubeconfig config get-contexts
```
```
CURRENT   NAME   CLUSTER      AUTHINFO   NAMESPACE
*         dev    kubernetes   cka-dev      
```

Because user `cka-dev` does not have authorization in the cluster, we will receive `forbidden` error when we try to get information of Pods or Nodes.
```console
kubectl --kubeconfig=/etc/kubernetes/pki/cka-dev.kubeconfig get pod 
kubectl --kubeconfig=/etc/kubernetes/pki/cka-dev.kubeconfig get node
```


#### Merge kubeconfig files

Make a copy of your existing config
```console
cp ~/.kube/config ~/.kube/config.old 
```

Merge the two config files together into a new config file `/tmp/config`.
```console
KUBECONFIG=~/.kube/config:/etc/kubernetes/pki/cka-dev.kubeconfig  kubectl config view --flatten > /tmp/config
```

Replace the old config with the new merged config
```console
mv /tmp/config ~/.kube/config
```

Now the new `~/.kube/config` looks like below.
```console
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <your_key>
    server: https://<cka001_ip>:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: cka-dev
  name: dev
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: cka-dev
  user:
    client-certificate-data: <your_key>
    client-key-data: <your_key>
- name: kubernetes-admin
  user:
    client-certificate-data: <your_key>
    client-key-data: <your_key>
```



Verify contexts after kubeconfig merged.
```console
kubectl config get-contexts
```
Current context is the system default `kubernetes-admin@kubernetes`.
```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          dev                           kubernetes   cka-dev            
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```



### Namespaces & Contexts

Get list of Namespace with Label information.
```console
kubectl get ns --show-labels
```

Create Namespace `cka`.
```console
kubectl create namespace cka
```

Use below command to set a context with new update, e.g, update default namespace, etc..
```console
kubectl config set-context <context name> --cluster=<cluster name> --namespace=<namespace name> --user=<user name> 
```

Let's set default namespace to each context.
```console
kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --namespace=default --user=kubernetes-admin
kubectl config set-context dev --cluster=kubernetes --namespace=cka --user=cka-dev
```

Let's check current context information.
```console
kubectl config get-contexts
```
Output:
```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          dev                           kubernetes   cka-dev            cka
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```

To switch to a new context, use below command.
```console
kubectl config use-contexts <context name>
```

For example.
```console
kubectl config use-context dev
```

Verify if it's changed as expected.
```console
kubectl config get-contexts
```
```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         dev            kubernetes   cka-dev            cka
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   dev
```

Be noted, four users beginning with `cka-dev` created don't have any authorizations, e.g., access namespaces, get pods, etc..
Referring RBAC to grant their authorizations. 






### Role & RoleBinding


Switch to context `kubernetes-admin@kubernetes`.
```console
kubectl config use-context kubernetes-admin@kubernetes
```

Use `kubectl create role` command  with option `--dry-run=client` and `-o yaml` to generate yaml template for customizing. 
```console
kubectl create role admin-dev --resource=pods --verb=get --verb=list --verb=watch --dry-run=client -o yaml
```

Create role `admin-dev` on namespace `cka`.
```console
kubectl apply -f - << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: cka
  name: admin-dev
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
EOF
```

Use `kubectl create rolebinding` command  with option `--dry-run=client` and `-o yaml` to generate yaml template for customizing.
```console
kubectl create rolebinding admin --role=admin-dev --user=cka-dev --dry-run=client -o yaml
```

Create rolebinding `admin` on namespace `cka`.
```console
kubectl apply -f - << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin
  namespace: cka
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: admin-dev
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: cka-dev
EOF
```

Verify authorzation of user `cka-dev` on Namespace `cka`.

Switch to context `dev`.
```console
kubectl config use-context dev
```



Get Pods status in Namespace `cka`. Success!
```console
kubectl get pod -n cka
```

Get Pods status in Namespace `kube-system`. Failed, because the authorzation is only for Namespace `cka`.
```console
kubectl get pod -n kube-system
```

Get Nodes status. Failed, because the role we defined is only for Pod resource.
```console
kubectl get node
```

Create a Pod in Namespace `dev`. Failed because we only have `get`,`watch`,`list` for Pod, no `create` authorization.
```console
kubectl run nginx --image=nginx -n cka
```





### ClusterRole & ClusterRoleBinding

Switch to context `kubernetes-admin@kubernetes`.
```console
kubectl config use-context kubernetes-admin@kubernetes
```

Create a ClusterRole `nodes-admin` with authorization `get`,`watch`,`list` for `nodes` resource.
```console
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nodes-admin
rules:
- apiGroups:
  - ""
  resources: 
  - nodes
  verbs:
  - get
  - watch
  - list
EOF
```

Bind ClusterRole `nodes-admin` to user `cka-dev`.
```console
kubectl apply -f - << EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
subjects:
- kind: User
  name: cka-dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: nodes-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

Verify Authorization

Switch to context `dev`.
```console
kubectl config use-context dev
```

Get node information. Success!
```console
kubectl get node
```

Switch to system context.
```console
kubectl config use-context kubernetes-admin@kubernetes 
```



### ClusterRole and ServiceAccount

!!! Scenario
    * Create a ClusterRole, which is authorized to create Deployment, StatefulSet, DaemonSet.
    * Bind the ClusterRole to a ServiceAccount.

Demo:

```console
kubectl create namespace my-namespace

kubectl -n my-namespace create serviceaccount my-sa

kubectl create clusterrole my-clusterrole --verb=create --resource=deployments,statefulsets,daemonsets

kubectl -n my-namespace create rolebinding my-clusterrolebinding --clusterrole=my-clusterrole --serviceaccount=my-namespace:my-sa
```

Clean up.
```console
kubectl delete namespace my-namespace 
kubectl delete clusterrole my-clusterrole
```

!!! Hints
    1. A RoleBinding may reference any Role in the same namespace. 
    2. A RoleBinding can reference a ClusterRole and bind that ClusterRole to the namespace of the RoleBinding. 
    3. If you want to bind a ClusterRole to all the namespaces in your cluster, you use a ClusterRoleBinding.
    4. Use RoleBinding to bind ClusterRole is to reuse the ClusterRole for namespaced resources, avoid duplicated namespaced roles for same authorization.


