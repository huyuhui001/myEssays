# Case Study: Install Calico

!!! Scenario
    Install Calico

    * Calico Datastore
    * Configure IP Pools
    * Install CNI plugin
    * Install Typha
    * Install calico/node
    * Test networking


## The Calico Datastore

In order to use Kubernetes as the Calico datastore, we need to define the custom resources Calico uses.

Download and examine the list of Calico custom resource definitions, and open it in a file editor.
```console
wget https://projectcalico.docs.tigera.io/manifests/crds.yaml
```

Create the custom resource definitions in Kubernetes.
```console
kubectl apply -f crds.yaml
```

Install `calicoctl`. To interact directly with the Calico datastore, use the `calicoctl` client tool.

Download the calicoctl binary to a Linux host with access to Kubernetes. 
The latest release of calicoctl can be found in the [git page](https://github.com/projectcalico/calico/releases) and replace below `v3.23.2` by actual release number.
```console
wget https://github.com/projectcalico/calico/releases/download/v3.23.3/calicoctl-linux-amd64
chmod +x calicoctl-linux-amd64
sudo cp calicoctl-linux-amd64 /usr/local/bin/calicoctl
```

Configure calicoctl to access Kubernetes
```console
echo "export KUBECONFIG=/root/.kube/config" >> ~/.bashrc
echo "export DATASTORE_TYPE=kubernetes" >> ~/.bashrc

echo $KUBECONFIG
echo $DATASTORE_TYPE
```

Verify `calicoctl` can reach the datastore by running：
```console
calicoctl get nodes -o wide
```
Output similar to below:
```
NAME     ASN   IPV4   IPV6   
cka001                       
cka002                       
cka003  
```

Nodes are backed by the Kubernetes node object, so we should see names that match `kubectl get nodes`.
```console
kubectl get nodes -o wide
```
```
NAME     STATUS     ROLES                  AGE   VERSION   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cka001   NotReady   control-plane,master   23m   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
cka002   NotReady   <none>                 22m   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
cka003   NotReady   <none>                 21m   v1.24.0   Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
```



## Configure IP Pools

A workload is a container or VM that Calico handles the virtual networking for. 
In Kubernetes, workloads are pods. 
A workload endpoint is the virtual network interface a workload uses to connect to the Calico network.

IP pools are ranges of IP addresses that Calico uses for workload endpoints.

Get current IP pools in the cluster. So far, it's empty after fresh installation.
```console
calicoctl get ippools
```
```
NAME   CIDR   SELECTOR 
```

The Pod CIDR is `10.244.0.0/16` we specified via `kubeadm init`.

Let's create two IP pools for use in the cluster. Each pool can not have any overlaps.

* ipv4-ippool-1: `10.244.0.0/18`
* ipv4-ippool-2: `10.244.192.0/19`

```console
calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool-1
spec:
  cidr: 10.244.0.0/18
  ipipMode: Never
  natOutgoing: true
  disabled: false
  nodeSelector: all()
EOF
```
```console
calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool-2
spec:
  cidr: 10.244.192.0/19
  ipipMode: Never
  natOutgoing: true
  disabled: true
  nodeSelector: all()
EOF
```

IP pool now looks like below.
```console
calicoctl get ippools -o wide
```
```
NAME            CIDR              NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
ipv4-ippool-1   10.244.0.0/18     true   Never      Never       false      false              all()      
ipv4-ippool-2   10.244.192.0/19   true   Never      Never       true       false              all()     
```


## Install CNI plugin

* Provision Kubernetes user account for the plugin.

Kubernetes uses the Container Network Interface (CNI) to interact with networking providers like Calico. 
The Calico binary that presents this API to Kubernetes is called the CNI plugin and must be installed on every node in the Kubernetes cluster.

The CNI plugin interacts with the Kubernetes API server while creating pods, both to obtain additional information and to update the datastore with information about the pod.

On the Kubernetes *master* node, create a key for the CNI plugin to authenticate with and certificate signing request.

Change to directory `/etc/kubernetes/pki/`.
```console
cd /etc/kubernetes/pki/
```
```
openssl req -newkey rsa:4096 \
  -keyout cni.key \
  -nodes \
  -out cni.csr \
  -subj "/CN=calico-cni"
```

We will sign this certificate using the main Kubernetes CA.
```console
sudo openssl x509 -req -in cni.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out cni.crt \
  -days 3650
```
Output looks like below. User is `calico-cni`.
```
Signature ok
subject=CN = calico-cni
Getting CA Private Key
```
```console
sudo chown $(id -u):$(id -g) cni.crt
```

Next, we create a kubeconfig file for the CNI plugin to use to access Kubernetes. 
Copy this `cni.kubeconfig` file to every node in the cluster.

Stay in directory `/etc/kubernetes/pki/`.
```console
APISERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')

echo $APISERVER

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=$APISERVER \
  --kubeconfig=cni.kubeconfig

kubectl config set-credentials calico-cni \
  --client-certificate=cni.crt \
  --client-key=cni.key \
  --embed-certs=true \
  --kubeconfig=cni.kubeconfig

kubectl config set-context cni@kubernetes \
  --cluster=kubernetes \
  --user=calico-cni \
  --kubeconfig=cni.kubeconfig

kubectl config use-context cni@kubernetes --kubeconfig=cni.kubeconfig
```

The context for CNI looks like below.
```console
kubectl config get-contexts --kubeconfig=cni.kubeconfig
```
```
CURRENT   NAME             CLUSTER      AUTHINFO     NAMESPACE
*         cni@kubernetes   kubernetes   calico-cni 
```



* Provision RBAC

Change to home directory
```console
cd ~
```

Define a cluster role the CNI plugin will use to access Kubernetes.

```console
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-cni
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
 # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
      - clusterinformations
      - ippools
    verbs:
      - get
      - list
EOF
```

Bind the cluster role to the `calico-cni` account.
```console
kubectl create clusterrolebinding calico-cni --clusterrole=calico-cni --user=calico-cni
```



* Install the plugin

Do these steps on **each node** in your cluster.

Installation on `cka001`.

Run these commands as **root**.
```console
sudo su
```

Install the CNI plugin Binaries. 
Get right release in the link `https://github.com/projectcalico/cni-plugin/releases`, and link `https://github.com/containernetworking/plugins/releases`.
```console
mkdir -p /opt/cni/bin

curl -L -o /opt/cni/bin/calico https://github.com/projectcalico/cni-plugin/releases/download/v3.20.5/calico-amd64
chmod 755 /opt/cni/bin/calico

curl -L -o /opt/cni/bin/calico-ipam https://github.com/projectcalico/cni-plugin/releases/download/v3.20.5/calico-ipam-amd64
chmod 755 /opt/cni/bin/calico-ipam
```
```console
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
tar xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin
```

Create the config directory
```console
mkdir -p /etc/cni/net.d/
```

Copy the kubeconfig from the previous section
```console
cp /etc/kubernetes/pki/cni.kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```

Write the CNI configuration
```console
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```
```console
cp /etc/cni/net.d/calico-kubeconfig ~
```

Exit from su and go back to the logged in user.
```console
exit
```


Installation on `cka002`.

```console
sftp -i cka-key-pair.pem cka002
```
```console
put calico-amd64
put calicoctl-linux-amd64
put calico-ipam-amd64
put calico-kubeconfig
put cni-plugins-linux-amd64-v1.1.1.tgz
```
```console
ssh -i cka-key-pair.pem cka002
```
```console
mkdir -p /opt/cni/bin

cp calico-amd64 /opt/cni/bin/calico
cp calico-ipam-amd64 /opt/cni/bin/calico-ipam

tar xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin

mkdir -p /etc/cni/net.d/

cp calico-kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```
```console
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

Back to `cka001`.
```console
exit
```


Installation on `cka003`.

```console
sftp -i cka-key-pair.pem cka003
```
```console
put calico-amd64
put calicoctl-linux-amd64
put calico-ipam-amd64
put calico-kubeconfig
put cni-plugins-linux-amd64-v1.1.1.tgz
```

```console
ssh -i cka-key-pair.pem cka003
```
```console
mkdir -p /opt/cni/bin

cp calico-amd64 /opt/cni/bin/calico
cp calico-ipam-amd64 /opt/cni/bin/calico-ipam

tar xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin

mkdir -p /etc/cni/net.d/

cp calico-kubeconfig /etc/cni/net.d/calico-kubeconfig

chmod 600 /etc/cni/net.d/calico-kubeconfig
```
```console
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

Back to `cka001`.
```console
exit
```

Stay in home directory in node `cka001`.

At this point Kubernetes nodes will become Ready because Kubernetes has a networking provider and configuration installed.
```console
kubectl get nodes
```
Result
```
NAME     STATUS   ROLES                  AGE     VERSION
cka001   Ready    control-plane,master   4h50m   v1.24.0
cka002   Ready    <none>                 4h49m   v1.24.0
cka003   Ready    <none>                 4h49m   v1.24.0
```






## Install Typha

Typha sits between the Kubernetes API server and per-node daemons like Felix and confd (running in calico/node). 
It watches the Kubernetes resources and Calico custom resources used by these daemons, and whenever a resource changes it fans out the update to the daemons. 
This reduces the number of watches the Kubernetes API server needs to serve and improves scalability of the cluster.

* Provision Certificates

We will use mutually authenticated TLS to ensure that calico/node and Typha communicate securely. 
We generate a certificate authority (CA) and use it to sign a certificate for Typha.

Change to directory `/etc/kubernetes/pki/`.
```console
cd /etc/kubernetes/pki/
```

Create the CA certificate and key
```console
openssl req -x509 -newkey rsa:4096 \
  -keyout typhaca.key \
  -nodes \
  -out typhaca.crt \
  -subj "/CN=Calico Typha CA" \
  -days 365
```

Store the CA certificate in a ConfigMap that Typha & calico/node will access.
```console
kubectl create configmap -n kube-system calico-typha-ca --from-file=typhaca.crt
```

Create the Typha key and certificate signing request (CSR).
```console
openssl req -newkey rsa:4096 \
  -keyout typha.key \
  -nodes \
  -out typha.csr \
  -subj "/CN=calico-typha"
```

The certificate presents the Common Name (CN) as `calico-typha`. `calico/node` will be configured to verify this name.

Sign the Typha certificate with the CA.
```console
openssl x509 -req -in typha.csr \
  -CA typhaca.crt \
  -CAkey typhaca.key \
  -CAcreateserial \
  -out typha.crt \
  -days 365
```
```
Signature ok
subject=CN = calico-typha
Getting CA Private Key
```

Store the Typha key and certificate in a secret that Typha will access
```console
kubectl create secret generic -n kube-system calico-typha-certs --from-file=typha.key --from-file=typha.crt
```


* Provision RBAC

Change to home directory.
```console
cd ~
```

Create a ServiceAccount that will be used to run Typha.
```console
kubectl create serviceaccount -n kube-system calico-typha
```

Define a cluster role for Typha with permission to watch Calico datastore objects.
```console
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-typha
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
      - endpoints
      - services
      - nodes
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - globalfelixconfigs
      - felixconfigurations
      - bgppeers
      - globalbgpconfigs
      - bgpconfigurations
      - ippools
      - ipamblocks
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - clusterinformations
      - hostendpoints
      - blockaffinities
      - networksets
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      #- ippools
      #- felixconfigurations
      - clusterinformations
    verbs:
      - get
      - create
      - update
EOF
```

Bind the cluster role to the calico-typha ServiceAccount.
```console
kubectl create clusterrolebinding calico-typha --clusterrole=calico-typha --serviceaccount=kube-system:calico-typha
```



* Install Deployment

Since Typha is required by `calico/node`, and `calico/node` establishes the pod network, we run Typha as a host networked pod to avoid a chicken-and-egg problem. 
We run 3 replicas of Typha so that even during a rolling update, a single failure does not make Typha unavailable.
```console
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      k8s-app: calico-typha
  template:
    metadata:
      labels:
        k8s-app: calico-typha
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: 'true'
    spec:
      hostNetwork: true
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
      serviceAccountName: calico-typha
      priorityClassName: system-cluster-critical
      containers:
      - image: calico/typha:v3.8.0
        name: calico-typha
        ports:
        - containerPort: 5473
          name: calico-typha
          protocol: TCP
        env:
          # Disable logging to file and syslog since those don't make sense in Kubernetes.
          - name: TYPHA_LOGFILEPATH
            value: "none"
          - name: TYPHA_LOGSEVERITYSYS
            value: "none"
          # Monitor the Kubernetes API to find the number of running instances and rebalance
          # connections.
          - name: TYPHA_CONNECTIONREBALANCINGMODE
            value: "kubernetes"
          - name: TYPHA_DATASTORETYPE
            value: "kubernetes"
          - name: TYPHA_HEALTHENABLED
            value: "true"
          # Location of the CA bundle Typha uses to authenticate calico/node; volume mount
          - name: TYPHA_CAFILE
            value: /calico-typha-ca/typhaca.crt
          # Common name on the calico/node certificate
          - name: TYPHA_CLIENTCN
            value: calico-node
          # Location of the server certificate for Typha; volume mount
          - name: TYPHA_SERVERCERTFILE
            value: /calico-typha-certs/typha.crt
          # Location of the server certificate key for Typha; volume mount
          - name: TYPHA_SERVERKEYFILE
            value: /calico-typha-certs/typha.key
        livenessProbe:
          httpGet:
            path: /liveness
            port: 9098
            host: localhost
          periodSeconds: 30
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /readiness
            port: 9098
            host: localhost
          periodSeconds: 10
        volumeMounts:
        - name: calico-typha-ca
          mountPath: "/calico-typha-ca"
          readOnly: true
        - name: calico-typha-certs
          mountPath: "/calico-typha-certs"
          readOnly: true
      volumes:
      - name: calico-typha-ca
        configMap:
          name: calico-typha-ca
      - name: calico-typha-certs
        secret:
          secretName: calico-typha-certs
EOF
```

We set `TYPHA_CLIENTCN` to calico-node which is the common name we will use on the certificate `calico/node` will use late.

Verify Typha is up an running with three instances
```console
kubectl get pods -l k8s-app=calico-typha -n kube-system
```
Result looks like below.
```
NAME                           READY   STATUS    RESTARTS   AGE
calico-typha-5b8669646-b2xnq   1/1     Running   0          20s
calico-typha-5b8669646-q5glk   0/1     Pending   0          20s
calico-typha-5b8669646-rvv86   1/1     Running   0          20s
```

Here is an error message received:
```
0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't have free ports for the requested pod ports.
```




* Install Service

`calico/node` uses a Kubernetes Service to get load-balanced access to Typha.
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  ports:
    - port: 5473
      protocol: TCP
      targetPort: calico-typha
      name: calico-typha
  selector:
    k8s-app: calico-typha
EOF
```

Validate that Typha is using TLS.
```console
TYPHA_CLUSTERIP=$(kubectl get svc -n kube-system calico-typha -o jsonpath='{.spec.clusterIP}')
```
```console
curl https://$TYPHA_CLUSTERIP:5473 -v --cacert /etc/kubernetes/pki/typhaca.crt
```
Result
```
*   Trying 11.244.91.165:5473...
* TCP_NODELAY set
* Connected to 11.244.91.165 (11.244.91.165) port 5473 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/kubernetes/pki/typhaca.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS alert, bad certificate (554):
* error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
* Closing connection 0
curl: (35) error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
```

This demonstrates that Typha is presenting its TLS certificate and rejecting our connection because we do not present a certificate. 
We will later deploy calico/node with a certificate Typha will accept.





## Install calico/node

`calico/node` runs three daemons:

* Felix, the Calico per-node daemon
* BIRD, a daemon that speaks the BGP protocol to distribute routing information to other nodes
* confd, a daemon that watches the Calico datastore for config changes and updates BIRD’s config files


* Provision Certificates

Change to directory `/etc/kubernetes/pki/`.
```console
cd /etc/kubernetes/pki/
```

Create the key `calico/node` will use to authenticate with Typha and the certificate signing request (CSR)
```console
openssl req -newkey rsa:4096 \
  -keyout calico-node.key \
  -nodes \
  -out calico-node.csr \
  -subj "/CN=calico-node"
```

The certificate presents the Common Name (CN) as `calico-node`, which is what we configured Typha to accept in the last lab.

Sign the Felix certificate with the CA we created earlier.
```console
openssl x509 -req -in calico-node.csr \
  -CA typhaca.crt \
  -CAkey typhaca.key \
  -CAcreateserial \
  -out calico-node.crt \
  -days 365
```
```
Signature ok
subject=CN = calico-node
Getting CA Private Key
```

Store the key and certificate in a Secret that calico/node will access.
```console
kubectl create secret generic -n kube-system calico-node-certs --from-file=calico-node.key --from-file=calico-node.crt
```


* Provision RBAC

Change to home directory.
```console
cd ~
```

Create the ServiceAccount that calico/node will run as.
```console
kubectl create serviceaccount -n kube-system calico-node
```

Provision a cluster role with permissions to read and modify Calico datastore objects
```console
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-node
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # EndpointSlices are used for Service-based network policy rule
  # enforcement.
  - apiGroups: ["discovery.k8s.io"]
    resources:
      - endpointslices
    verbs:
      - watch
      - list
  - apiGroups: [""]
    resources:
      - endpoints
      - services
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
      # Used to discover Typhas.
      - get
  # Pod CIDR auto-detection on kubeadm needs access to config maps.
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      # Needed for clearing NodeNetworkUnavailable flag.
      - patch
      # Calico stores some configuration information in node annotations.
      - update
  # Watch for changes to Kubernetes NetworkPolicies.
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  # Used by Calico for policy information.
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
    verbs:
      - list
      - watch
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
  # Used for creating service account tokens to be used by the CNI plugin
  - apiGroups: [""]
    resources:
      - serviceaccounts/token
    resourceNames:
      - calico-node
    verbs:
      - create
  # Calico monitors various CRDs for config.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - globalfelixconfigs
      - felixconfigurations
      - bgppeers
      - globalbgpconfigs
      - bgpconfigurations
      - ippools
      - ipamblocks
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - networksets
      - clusterinformations
      - hostendpoints
      - blockaffinities
    verbs:
      - get
      - list
      - watch
  # Calico must create and update some CRDs on startup.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ippools
      - felixconfigurations
      - clusterinformations
    verbs:
      - create
      - update
  # Calico stores some configuration information on the node.
  - apiGroups: [""]
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
  # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
    verbs:
      - get
  # Block affinities must also be watchable by confd for route aggregation.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
    verbs:
      - watch
EOF
```

Bind the cluster role to the calico-node ServiceAccount
```console
kubectl create clusterrolebinding calico-node --clusterrole=calico-node --serviceaccount=kube-system:calico-node
```



* Install daemon set

Change to home directory.
```console
cd ~
```

`calico/node` runs as a daemon set so that it is installed on every node in the cluster.

Change `image: calico/node:v3.20.0` to right version. 

Create the daemon set
```console
kubectl apply -f - <<EOF
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      hostNetwork: true
      tolerations:
        # Make sure calico-node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-node
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      containers:
        # Runs calico-node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: calico/node:v3.20.0
          env:
            # Use Kubernetes API as the backing datastore.
            - name: DATASTORE_TYPE
              value: "kubernetes"
            - name: FELIX_TYPHAK8SSERVICENAME
              value: calico-typha
            # Wait for the datastore.
            - name: WAIT_FOR_DATASTORE
              value: "true"
            # Set based on the k8s node name.
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              value: bird
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            # Disable file logging so kubectl logs works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            - name: FELIX_HEALTHENABLED
              value: "true"
            # Location of the CA bundle Felix uses to authenticate Typha; volume mount
            - name: FELIX_TYPHACAFILE
              value: /calico-typha-ca/typhaca.crt
            # Common name on the Typha certificate; used to verify we are talking to an authentic typha
            - name: FELIX_TYPHACN
              value: calico-typha
            # Location of the client certificate for connecting to Typha; volume mount
            - name: FELIX_TYPHACERTFILE
              value: /calico-node-certs/calico-node.crt
            # Location of the client certificate key for connecting to Typha; volume mount
            - name: FELIX_TYPHAKEYFILE
              value: /calico-node-certs/calico-node.key
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          lifecycle:
            preStop:
              exec:
                command:
                - /bin/calico-node
                - -shutdown
          livenessProbe:
            httpGet:
              path: /liveness
              port: 9099
              host: localhost
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -bird-ready
              - -felix-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /run/xtables.lock
              name: xtables-lock
              readOnly: false
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /var/run/nodeagent
              name: policysync
            - mountPath: "/calico-typha-ca"
              name: calico-typha-ca
              readOnly: true
            - mountPath: /calico-node-certs
              name: calico-node-certs
              readOnly: true
      volumes:
        # Used by calico-node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
        # Used to create per-pod Unix Domain Sockets
        - name: policysync
          hostPath:
            type: DirectoryOrCreate
            path: /var/run/nodeagent
        - name: calico-typha-ca
          configMap:
            name: calico-typha-ca
        - name: calico-node-certs
          secret:
            secretName: calico-node-certs
EOF
```

Verify that calico/node is running on each node in your cluster, and goes to Running within a few minutes.
```console
kubectl get pod -l k8s-app=calico-node -n kube-system
```
Result looks like below.
```
NAME                READY   STATUS    RESTARTS   AGE
calico-node-4c4sp   1/1     Running   0          40s
calico-node-j2z6v   1/1     Running   0          40s
calico-node-vgm9n   1/1     Running   0          40s
```


## Test networking

### Pod to pod pings

Create three busybox instances
```console
kubectl create deployment pingtest --image=busybox --replicas=3 -- sleep infinity
```

Check their IP addresses
```console
kubectl get pods --selector=app=pingtest --output=wide
```
Result
```
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
pingtest-585b76c894-chwjq   1/1     Running   0          7s    10.244.31.1    cka002   <none>           <none>
pingtest-585b76c894-s2tbs   1/1     Running   0          7s    10.244.31.0    cka002   <none>           <none>
pingtest-585b76c894-vm9wn   1/1     Running   0          7s    10.244.28.64   cka003   <none>           <none>
```

Note the IP addresses of the second two pods, then exec into the first one. 
From inside the pod, ping the other two pod IP addresses. 
For example:
```console
kubectl exec -ti pingtest-585b76c894-chwjq -- sh
/ # ping 10.244.31.1 -c 4
4 packets transmitted, 4 packets received, 0% packet loss

/ # ping 10.244.31.0 -c 4
4 packets transmitted, 4 packets received, 0% packet loss

/ # ping 10.244.28.64 -c 4
4 packets transmitted, 0 packets received, 100% packet loss
```


### Check routes

From one of the nodes, verify that routes exist to each of the pingtest pods’ IP addresses. For example
```console
ip route get 10.244.31.1
ip route get 10.244.31.0
ip route get 10.244.28.64
```
In the result, the `via <cka001_ip>`(it's control-plane) in this example indicates the next-hop for this pod IP, which matches the IP address of the node the pod is scheduled on, as expected.
IPAM allocations from different pools.

Recall that we created two IP pools, but left one disabled.
```console
calicoctl get ippools -o wide
```
Result
```
NAME            CIDR              NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
ipv4-ippool-1   10.244.0.0/18     true   Never      Never       false      false              all()      
ipv4-ippool-2   10.244.192.0/19   true   Never      Never       true       false              all()   
```

Enable the second pool.
```console
calicoctl --allow-version-mismatch apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool-2
spec:
  cidr: 10.244.192.0/19
  ipipMode: Never
  natOutgoing: true
  disabled: false
  nodeSelector: all()
EOF
```

```console
calicoctl get ippools -o wide
```
Result
```
NAME            CIDR              NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
ipv4-ippool-1   10.244.0.0/18     true   Never      Never       false      false              all()      
ipv4-ippool-2   10.244.192.0/19   true   Never      Never       false      false              all()      
```


Create a pod, explicitly requesting an address from pool2
```console
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pingtest-ippool-2
  annotations:
    cni.projectcalico.org/ipv4pools: "[\"ipv4-ippool-2\"]"
spec:
  containers:
  - args:
    - sleep
    - infinity
    image: busybox
    imagePullPolicy: Always
    name: pingtest
EOF
```

Verify it has an IP address from pool2
```console
kubectl get pod pingtest-ippool-2 -o wide
```
Result
```
NAME                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
pingtest-ippool-2   1/1     Running   0          18s   10.244.203.192   cka003   <none>           <none>
```

Let's attach to the Pod `pingtest-585b76c894-chwjq` again.
```console
kubectl exec -ti pingtest-585b76c894-chwjq -- sh
/ # 10.244.203.192 -c 4
4 packets transmitted, 0 packets received, 100% packet loss
```

!! Mark here. it's failed. Need further check why the route does not work.

Clean up
```console
kubectl delete deployments.apps pingtest
kubectl delete pod pingtest-ippool-2
```



!!! Reference
    [End-to-end Calico installation](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/)
