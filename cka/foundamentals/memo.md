# CKA自学笔记5:Kubernetes基本概念散记

## Kubernetes基本概念

### Kubernetes Components

A Kubernetes cluster consists of the components that represent the **control plane** and a set of machines called **nodes**.

![The components of a Kubernetes cluster](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)


**Kubernetes Components**: 

* Control Plane Components
    * kube-apiserver: 
        * query and manipulate the state of objects in Kubernetes.
        * play as "communication hub" among all resources in cluster.
        * provide cluster security authentication, authorization, and role assignment.
        * the only one can connect to `etcd`.
    * etcd: 
        * all Kubernetes objects are stored on etcd. 
        * Kubernetes objects are persistent **entities** in the Kubernetes system, which are used to represent the state of your cluster.
    * kube-scheduler: 
        * watches for newly created Pods with no assigned node, and selects a node for them to run on.
    * kube-controller-manager: runs controller processes.
        * *Node controller*: Responsible for noticing and responding when nodes go down.
        * *Job controller*: Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion.
        * *Endpoints controller*: Populates the Endpoints object (that is, joins Services & Pods).
        * *Service Account & Token controllers*: Create default accounts and API access tokens for new namespaces.
    * cloud-controller-manager: embeds cloud-specific control logic and only runs controllers that are specific to your cloud provider, no need for own premises and learning environment.
        * *Node controller*: For checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding
        * *Route controller*: For setting up routes in the underlying cloud infrastructure
        * *Service controller*: For creating, updating and deleting cloud provider load balancers
* Node Components
    * kubelet: 
        * An agent that runs on each node in the cluster. 
        * Manage node. It makes sure that containers are running in a Pod. `kubelet` registers and updates nodes information to APIServer, and APIServer stores them into `etcd`.
        * Manage pod. Watch pod via APIServer, and action on pods or containers in pods.
        * Health check at container level.
    * kube-proxy: 
        * is a network proxy that runs on each node in cluster.
            * iptables
            * ipvs
        * maintains network rules on nodes.
    * Container runtime: 
        * is the software that is responsible for running containers.
* Addons
    * DNS: is a DNS server and required by all Kubernetes clusters.
    * Web UI (Dashboard): web-based UI for Kubernetes clusters. 
    * Container Resource Monitoring: records generic time-series metrics about containers in a central database
    * Cluster-level Logging: is responsible for saving container logs to a central log store with search/browsing interface.


Scalability:

* **Scaling out** (horizontal scaling) by adding more servers to your architecture to spread the workload across more machines.
* **Scaling up** (vertical scaling) by adding more hard drives and memory to increase the computing capacity of physical servers. 





### Kubernetes API 

The REST API is the fundamental fabric of Kubernetes. 
All operations and communications between components, and external user commands are REST API calls that the API Server handles. 
Consequently, everything in the Kubernetes platform is treated as an *API object* and has a corresponding entry in the API.

The core of Kubernetes' control plane is the API server. 

* CRI: Container Runtime Interface
* CNI: Container Network Interface
* CSI: Container Storage Interface

The API server exposes an HTTP API that lets end users, different parts of cluster, and external components communicate with one another.

The Kubernetes API lets we query and manipulate the state of API objects in Kubernetes (for example: Pods, Namespaces, ConfigMaps, and Events).

Kubernetes API:

* OpenAPI specification
    * OpenAPI V2
    * OpenAPI V3
* Persistence. Kubernetes stores the serialized state of objects by writing them into etcd.
* API groups and versioning. Versioning is done at the API level. API resources are distinguished by their API group, resource type, namespace (for namespaced resources), and name.
    * API changes
* API Extension



#### API Version

The API versioning and software versioning are indirectly related. 
The API and release versioning proposal describes the relationship between API versioning and software versioning.
Different API versions indicate different levels of stability and support. 

Here's a summary of each level:

* Alpha:
    * The version names contain alpha (for example, v1alpha1).
    * The software may contain bugs. Enabling a feature may expose bugs. A feature may be disabled by default.
    * The support for a feature may be dropped at any time without notice.
    * The API may change in incompatible ways in a later software release without notice.
    * The software is recommended for use only in short-lived testing clusters, due to increased risk of bugs and lack of long-term support.
* Beta:
    * The version names contain beta (for example, v2beta3).
    * The software is well tested. Enabling a feature is considered safe. Features are enabled by default.
    * The support for a feature will not be dropped, though the details may change.
    * The schema and/or semantics of objects may change in incompatible ways in a subsequent beta or stable release. When this happens, migration instructions are provided. Schema changes may require deleting, editing, and re-creating API objects. The editing process may not be straightforward. The migration may require downtime for applications that rely on the feature.
    * The software is not recommended for production uses. Subsequent releases may introduce incompatible changes. If you have multiple clusters which can be upgraded independently, you may be able to relax this restriction.
      Note: Please try beta features and provide feedback. After the features exit beta, it may not be practical to make more changes.
* Stable:
    * The version name is vX where X is an integer.
    * The stable versions of features appear in released software for many subsequent versions.


Command to get current API
```
kubectl api-resources
```

#### API Group

[API groups](https://git.k8s.io/design-proposals-archive/api-machinery/api-group.md) make it easier to extend the Kubernetes API. 
The API group is specified in a REST path and in the apiVersion field of a serialized object.

There are several API groups in Kubernetes:

* The core (also called legacy) group is found at REST path `/api/v1`. 
    * The core group is not specified as part of the apiVersion field, for example, apiVersion: v1.
* The named groups are at REST path `/apis/$GROUP_NAME/$VERSION` and use apiVersion: `$GROUP_NAME/$VERSION` (for example, apiVersion: batch/v1). 










### Kubernetes Objects

#### Objects Overview:

* Object Spec:
    * providing a description of the characteristics the resource created to have: *its desired state*.
* Object Status:
    * describes the current state of the object.

Example of Deployment as an object that can represent an application running on cluster.
```
apiVersion: apps/v1  # Which version of the Kubernetes API you're using to create this object
kind: Deployment     # What kind of object you want to create
metadata:            # Data that helps uniquely identify the object, including a name string, UID, and optional namespace
  name: nginx-deployment
spec:                # What state you desire for the object
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```


#### Object Management:

The `kubectl` command-line tool supports several different ways to create and manage Kubernetes objects. Read the [Kubectl book](https://kubectl.docs.kubernetes.io/) for details.

A Kubernetes object should be managed using ONLY one technique. Mixing and matching techniques for the same object results in undefined behavior. 

Three management techniques:

* Imperative commands
    * operates directly on live objects in a cluster. 
    * `kubectl create deployment nginx --image nginx`
* Imperative object configuration
    * `kubectl create -f nginx.yaml`
    * `kubectl delete -f nginx.yaml -f redis.yaml`
    * `kubectl replace -f nginx.yaml`
* Declarative object configuration
    * `kubectl diff -f configs/`
    * `kubectl apply -f configs/`





#### Object Names and IDs

Each object in your cluster has a *Name* that is unique for that type of resource.

* DNS Subdomain Names
* Label Names
* Path Segment Names

Every Kubernetes object also has a *UID* that is unique across the whole cluster.





#### Namespaces

In Kubernetes, namespaces provides a mechanism for isolating groups of resources within a single cluster. 

Names of resources need to be unique within a namespace, but not across namespaces. 

Namespace-based scoping is applicable only for namespaced objects (e.g. Deployments, Services, etc) and not for cluster-wide objects (e.g. StorageClass, Nodes, PersistentVolumes, etc)

Not All Objects are in a Namespace.


Kubernetes starts with four initial namespaces:

* `default` 
    The default namespace for objects with no other namespace
* `kube-system` 
    The namespace for objects created by the Kubernetes system
* `kube-public` 
    This namespace is created automatically and is readable by all users (including those not authenticated). 
    This namespace is mostly reserved for cluster usage, in case that some resources should be visible and readable publicly throughout the whole cluster. 
    The public aspect of this namespace is only a convention, not a requirement.
* `kube-node-lease` This namespace holds Lease objects associated with each node. Node leases allow the kubelet to send heartbeats so that the control plane can detect node failure.


Viewing namespaces: 

* `kubectl get namespace`

Setting the namespace for a request

* `kubectl run nginx --image=nginx --namespace=<insert-namespace-name-here>`
* `kubectl get pods --namespace=<insert-namespace-name-here>`






#### Labels and Selectors

Labels are key/value pairs that are attached to objects, such as pods. 
Valid label keys have two segments: an optional prefix and name, separated by a slash (`/`).

Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users.

Labels can be used to organize and to select subsets of objects. 
Labels can be attached to objects at creation time and subsequently added and modified at any time. 
Each object can have a set of key/value labels defined. 
Each Key must be unique for a given object.

Example of labels:
```
"metadata": {
    "labels": {
        "key1" : "value1",
        "key2" : "value2"
    }
}
```


Unlike names and UIDs, labels do not provide uniqueness. In general, we expect many objects to carry the same label(s).

The API currently supports two types of selectors: 

* equality-based, e.g., `environment = production`, `tier != frontend`
* set-based, e.g., `environment in (production, qa)`, `tier notin (frontend, backend)`

Sample commands:
```
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production),tier in (frontend)'
kubectl get pods -l 'environment in (production, qa)'
kubectl get pods -l 'environment,environment notin (frontend)'
```





#### Annotations

Use Kubernetes annotations to attach arbitrary non-identifying metadata to objects. 
Clients such as tools and libraries can retrieve this metadata.

Use either labels or annotations to attach metadata to Kubernetes objects. 

* Labels can be used to select objects and to find collections of objects that satisfy certain conditions. 
* Annotations are not used to identify and select objects. 


Annotations, like labels, are key/value maps. The keys and the values in the map must be strings. 
```
"metadata": {
    "annotations": {
      "key1" : "value1",
      "key2" : "value2"
    }
}
```

Valid annotation keys have two segments: an optional prefix and name, separated by a slash (`/`). 






#### Field Selectors

Field selectors let you select Kubernetes resources based on the value of one or more resource fields. 

Here are some examples of field selector queries:
```
metadata.name=my-service
metadata.namespace!=default
status.phase=Pending
```

This kubectl command selects all Pods for which the value of the status.phase field is Running:
`kubectl get pods --field-selector status.phase=Running`


Supported field selectors vary by Kubernetes resource type. All resource types support the `metadata.name` and `metadata.namespace` fields. 

Use the `=`, `==`, and `!=` operators with field selectors (`=` and `==` mean the same thing). 

For example:

`kubectl get ingress --field-selector foo.bar=baz`

With operators, 
`kubectl get services  --all-namespaces --field-selector metadata.namespace!=default`

Chained selectors, 
`kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always`

Multiple resource types, 
`kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default`






#### Finalizers

Finalizers are *namespaced keys* that tell Kubernetes to wait until specific conditions are met before it fully deletes resources marked for *deletion*. 
*Finalizers alert controllers* to clean up resources the deleted object owned.

Finalizers are usually added to resources for a reason, so forcefully removing them can lead to issues in the cluster.

Like labels, *owner references* describe the relationships between objects in Kubernetes, but are used for a different purpose.

Kubernetes uses the owner references (not labels) to determine which Pods in the cluster need cleanup.

Kubernetes processes finalizers when it identifies owner references on a resource targeted for deletion.


#### Owners and Dependents

In Kubernetes, some objects are owners of other objects. For example, a ReplicaSet is the owner of a set of Pods. 
These owned objects are dependents of their owner.

Dependent objects have a `metadata.ownerReferences` field that references their owner object.

A valid owner reference consists of the object name and a UID within the same namespace as the dependent object.

Dependent objects also have an `ownerReferences.blockOwnerDeletion` field that takes a boolean value and controls whether specific dependents can block garbage collection from deleting their owner object. 





### Resource

Kubernetes resources and "records of intent" are all stored as API objects, and modified via RESTful calls to the API. 
The API allows configuration to be managed in a declarative way. 
Users can interact with the Kubernetes API directly, or via tools like kubectl. 
The core Kubernetes API is flexible and can also be extended to support custom resources.

* Workload Resources
    * *Pod*. Pod is a collection of containers that can run on a host.
    * *PodTemplate*. PodTemplate describes a template for creating copies of a predefined pod.
    * *ReplicationController*. ReplicationController represents the configuration of a replication controller.
    * *ReplicaSet*. ReplicaSet ensures that a specified number of pod replicas are running at any given time.
    * *Deployment*. Deployment enables declarative updates for Pods and ReplicaSets.
    * *StatefulSet*. StatefulSet represents a set of pods with consistent identities.
    * *ControllerRevision*. ControllerRevision implements an immutable snapshot of state data.
    * *DaemonSet*. DaemonSet represents the configuration of a daemon set.
    * *Job*. Job represents the configuration of a single job.
    * *CronJob*. CronJob represents the configuration of a single cron job.
    * *HorizontalPodAutoscaler*. configuration of a horizontal pod autoscaler.
    * *HorizontalPodAutoscaler*. HorizontalPodAutoscaler is the configuration for a horizontal pod autoscaler, which automatically manages the replica count of any resource implementing the scale subresource based on the metrics specified.
    * *HorizontalPodAutoscaler v2beta2*. HorizontalPodAutoscaler is the configuration for a horizontal pod autoscaler, which automatically manages the replica count of any resource implementing the scale subresource based on the metrics specified.
    * *PriorityClass*. PriorityClass defines mapping from a priority class name to the priority integer value.
* Service Resources
    * *Service*. Service is a named abstraction of software service (for example, mysql) consisting of local port (for example 3306) that the proxy listens on, and the selector that determines which pods will answer requests sent through the proxy.
    * *Endpoints*. Endpoints is a collection of endpoints that implement the actual service.
    * *EndpointSlice*. EndpointSlice represents a subset of the endpoints that implement a service.
    * *Ingress*. Ingress is a collection of rules that allow inbound connections to reach the endpoints defined by a backend.
    * *IngressClass*. IngressClass represents the class of the Ingress, referenced by the Ingress Spec.
* Config and Storage Resources
    * *ConfigMap*. ConfigMap holds configuration data for pods to consume.
    * *Secret*. Secret holds secret data of a certain type.
    * *Volume*. Volume represents a named volume in a pod that may be accessed by any container in the pod.
    * *PersistentVolumeClaim*. PersistentVolumeClaim is a user's request for and claim to a persistent volume.
    * *PersistentVolume*. PersistentVolume (PV) is a storage resource provisioned by an administrator.
    * *StorageClass*. StorageClass describes the parameters for a class of storage for which PersistentVolumes can be dynamically provisioned.
    * *VolumeAttachment*. VolumeAttachment captures the intent to attach or detach the specified volume to/from the specified node.
    * *CSIDriver*. CSIDriver captures information about a Container Storage Interface (CSI) volume driver deployed on the cluster.
    * *CSINode*. CSINode holds information about all CSI drivers installed on a node.
    * *CSIStorageCapacity*. CSIStorageCapacity stores the result of one CSI GetCapacity call.
* Authentication Resources
    * *ServiceAccount*. ServiceAccount binds together: 
        * a name, understood by users, and perhaps by peripheral systems, for an identity 
        * a principal that can be authenticated and authorized 
        * a set of secrets.
    * *TokenRequest*. TokenRequest requests a token for a given service account.
    * *TokenReview*. TokenReview attempts to authenticate a token to a known user.
    * *CertificateSigningRequest*. CertificateSigningRequest objects provide a mechanism to obtain x509 certificates by submitting a certificate signing request, and having it asynchronously approved and issued.
* Authorization Resources
    * *LocalSubjectAccessReview*. LocalSubjectAccessReview checks whether or not a user or group can perform an action in a given namespace.
    * *SelfSubjectAccessReview*. SelfSubjectAccessReview checks whether or the current user can perform an action.
    * *SelfSubjectRulesReview*. SelfSubjectRulesReview enumerates the set of actions the current user can perform within a namespace.
    * *SubjectAccessReview*. SubjectAccessReview checks whether or not a user or group can perform an action.
    * *ClusterRole*. ClusterRole is a cluster level, logical grouping of PolicyRules that can be referenced as a unit by a RoleBinding or ClusterRoleBinding.
    * *ClusterRoleBinding*. ClusterRoleBinding references a ClusterRole, but not contain it.
    * *Role*. Role is a namespaced, logical grouping of PolicyRules that can be referenced as a unit by a RoleBinding.
    * *RoleBinding*. RoleBinding references a role, but does not contain it.
* Policy Resources
    * *LimitRange*. LimitRange sets resource usage limits for each kind of resource in a Namespace.
    * *ResourceQuota*. ResourceQuota sets aggregate quota restrictions enforced per namespace.
    * *NetworkPolicy*. NetworkPolicy describes what network traffic is allowed for a set of Pods.
    * *PodDisruptionBudget*. PodDisruptionBudget is an object to define the max disruption that can be caused to a collection of pods.
    * *PodSecurityPolicy v1beta1*. PodSecurityPolicy governs the ability to make requests that affect the Security Context that will be applied to a pod and container.
* Extend Resources
    * *CustomResourceDefinition*. CustomResourceDefinition represents a resource that should be exposed on the API server.
    * *MutatingWebhookConfiguration*. MutatingWebhookConfiguration describes the configuration of and admission webhook that accept or reject and may change the object.
    * *ValidatingWebhookConfiguration(). ValidatingWebhookConfiguration describes the configuration of and admission webhook that accept or reject and object without changing it.
* Cluster Resources
    * *Node*. Node is a worker node in Kubernetes.
    * *Namespace*. Namespace provides a scope for Names.
    * *Event*. Event is a report of an event somewhere in the cluster.
    * *APIService*. APIService represents a server for a particular GroupVersion.
    * *Lease*. Lease defines a lease concept.
    * *RuntimeClass*. RuntimeClass defines a class of container runtime supported in the cluster.
    * *FlowSchema v1beta2*. FlowSchema defines the schema of a group of flows.
    * *PriorityLevelConfiguration v1beta2*. PriorityLevelConfiguration represents the configuration of a priority level.
    * *Binding*. Binding ties one object to another; for example, a pod is bound to a node by a scheduler.
    * *ComponentStatus*. ComponentStatus (and ComponentStatusList) holds the cluster validation info.


Command `kube api-resources` to get the supported API resources.


Command `kubectl explain RESOURCE [options]` describes the fields associated with each supported API resource. 
Fields are identified via a simple JSONPath identifier:
```
kubectl explain binding
kubectl explain binding.metadata
kubectl explain binding.metadata.name
```



## Workload Resources

### Pods

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

A Pod is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers.

A Pod's contents are always co-located and co-scheduled, and run in a shared context. 

A Pod models an application-specific "logical host": it contains one or more application containers which are relatively tightly coupled. 

In non-cloud contexts, applications executed on the same physical or virtual machine are analogous to cloud applications executed on the same logical host.

The shared context of a Pod is a set of Linux namespaces, cgroups, and potentially other facets of isolation - the same things that isolate a Docker container.

In terms of Docker concepts, a Pod is similar to a group of Docker containers with shared namespaces and shared filesystem volumes.

Usually you don't need to create Pods directly, even singleton Pods. Instead, create them using workload resources such as *Deployment* or *Job*. 
If your Pods need to track state, consider the StatefulSet resource.

Pods in a Kubernetes cluster are used in two main ways:

* Pods that run a single container. 
* Pods that run multiple containers that need to work together. 

The "one-container-per-Pod" model is the most common Kubernetes use case; 
in this case, you can think of a Pod as a wrapper around a single container; 
Kubernetes manages Pods rather than managing the containers directly.

A Pod can encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. 

These co-located containers form a single cohesive unit of service—for example, one container serving data stored in a shared volume to the public, 
while a separate sidecar container refreshes or updates those files. 
The Pod wraps these containers, storage resources, and an ephemeral network identity together as a single unit.

Grouping multiple co-located and co-managed containers in a single Pod is a relatively advanced use case. 
You should use this pattern *only* in specific instances in which your containers are tightly coupled.

Each Pod is meant to run a single instance of a given application. 
If you want to scale your application horizontally (to provide more overall resources by running more instances), you should use multiple Pods, one for each instance. 
In Kubernetes, this is typically referred to as *replication*. Replicated Pods are usually created and managed as a group by a workload resource and its controller.

Pods natively provide two kinds of shared resources for their constituent containers: *[networking](https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking)* and *[storage](https://kubernetes.io/docs/concepts/workloads/pods/#pod-storage)*.

A Pod can specify a set of shared storage volumes. All containers in the Pod can access the shared volumes, allowing those containers to share data. 

Each Pod is assigned a unique IP address for each address family.
Within a Pod, containers share an IP address and port space, and can find each other via `localhost`.
Containers that want to interact with a container running in a different Pod can use IP networking to communicate.

When a Pod gets created, the new Pod is scheduled to run on a Node in your cluster. 
The Pod remains on that node until the Pod finishes execution, the Pod object is deleted, the Pod is evicted for lack of resources, or the node fails.

Restarting a container in a Pod should not be confused with restarting a Pod. 
A Pod is not a process, but an environment for running container(s). 
A Pod persists until it is deleted.

You can use workload resources (e.g., Deployment, StatefulSet, DaemonSet) to create and manage multiple Pods for you. 
A controller for the resource handles replication and rollout and automatic healing in case of Pod failure.


![Pod with multiple containers](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)


#### InitContainer

Some Pods have init containers as well as app containers. Init containers run and complete before the app containers are started.

You can specify init containers in the Pod specification alongside the containers array (which describes app containers).


#### Static Pod

Static Pods are managed directly by the kubelet daemon on a specific node, without the API server observing them. 

Static Pods are always bound to one Kubelet on a specific node. 

The main use for static Pods is to run a self-hosted control plane: in other words, using the kubelet to supervise the individual control plane components.

The kubelet automatically tries to create a mirror Pod on the Kubernetes API server for each static Pod. 
This means that the Pods running on a node are visible on the API server, but cannot be controlled from there.


#### Container probes

A probe is a diagnostic performed periodically by the kubelet on a container. 

To perform a diagnostic, the kubelet either executes code within the container, or makes a network request.

There are four different ways to check a container using a probe. Each probe must define exactly one of these four mechanisms:

* *exec*. The diagnostic is considered successful if the command exits with a status code of 0.
* *grpc*. The diagnostic is considered successful if the status of the response is SERVING.
* *httpGet*. The diagnostic is considered successful if the response has a status code greater than or equal to 200 and less than 400.
* *tcpSocket*. The diagnostic is considered successful if the port is open.

Each probe has one of three results:

* Success
* Failure
* Unknown

Types of probe:

* *livenessProbe*. Indicates whether the container is running. 
* *readinessProbe*. Indicates whether the container is ready to respond to requests.
* *startupProbe*. Indicates whether the application within the container is started.





### Deployment


### ReplicaSet

A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time. 
As such, it is often used to guarantee the availability of a specified number of identical Pods.

You may never need to manipulate ReplicaSet objects: use a Deployment instead, and define your application in the spec section.

You can specify how many Pods should run concurrently by setting `replicaset.spec.replicas`. 
The ReplicaSet will create/delete its Pods to match this number.
If you do not specify `replicaset.spec.replicas`, then it defaults to `1`.



### StatefulSet

StatefulSet Characteristics (aka, stick ID):

* Pod's name is immutable after created.
* DNS hostname is immutable after created.
* Mounted volume is immutable after created.

Stick ID of StatefulSet won't be changed after failure, scaling, and other operations. 

Naming convention of StatefulSet: `<StatefulSetName>-<Integer>`.

StatefulSet can be scalling by itsself, but Deployment need rely on ReplicaSet for scalling.

Recommendation: reduce StatefulSet to 0 first instead of delete it directly.


*headless* Service and *governing* Service:

* Headless Service is a normal Kubernetes Service object that its spec.clusterIP is set to `None`.
* When `spec.ServiceName` of StatefulSet is set to the headless Service name, the StatefulSet is now a governing Service.


General procedure to create a StatefulSet: 

* Create a StorageClass
* Create Headless Service
* Create StatefulSet based on above two.






### DaemonSet

A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are removed from the cluster, those Pods are garbage collected. 

Deleting a DaemonSet will clean up the Pods it created.

Some typical uses of a DaemonSet are:

* running a cluster storage daemon on every node
* running a logs collection daemon on every node
* running a node monitoring daemon on every node

In a simple case, one DaemonSet, covering all nodes, would be used for each type of daemon. 

A more complex setup might use multiple DaemonSets for a single type of daemon, but with different flags and/or different memory and cpu requests for different hardware types.

The DaemonSet controller reconciliation process reviews both existing nodes and newly created nodes. 

By default, the Kubernetes scheduler ignores the pods created by the DamonSet, and lets them exist on the node until the node itself is shut down. 

Running Pods on select Nodes:

* If you specify a `daemonset.spec.template.spec.nodeSelector`, then the DaemonSet controller will create Pods on nodes which match that node selector. 
* If you specify a `daemonset.spec.template.spec.affinity`, then DaemonSet controller will create Pods on nodes which match that node affinity. 
* If you do not specify either, then the DaemonSet controller will create Pods on all nodes.

There is no field `replicas` in `kubectl explain daemonset.spec` against with `kubectl explain deployment.spec.replicas`.
When a DaemonSet is created, each node will have *one* DaemonSet Pod running.

We’ll use a `Deployment`/`ReplicaSet` for services, mostly stateless, where we don’t care where the node is running, 
but we care more about the number of copies of our pod is running, and we can scale those copies/replicas up or down. 
Rolling updates would also be a benefit here.


We’ll use a `DaemonSet` when a copy of our pod must be running on the specific nodes that we require. 
Our daemon pod also needs to start before any of our other pods.


A DaemonSet is a simple scalability strategy for background services. 
When more eligible nodes are added to the cluster, the background service scales up. 
When nodes are removed, it will automatically scale down.









### Job





### CronJob





## Service Resource


### Service

Service is a named abstraction of software service (for example, mysql) consisting of local port (for example 3306) that the proxy listens on, and the selector that determines which pods will answer requests sent through the proxy.

The set of Pods targeted by a Service is usually determined by a selector (label selector). 

Type of service resource:

* ClusterIP Service (default): Reliable IP, DNS, and Port. Internal acess only.
* NodePort Service: Expose to external access.
* LoadBalancer: Based on NodePort and integrated with loader balance provided by cloud venders (e.g., AWS, GCP, etc.).
* ExternalName: Acces will be trafficed to external service.

Here is an example of yaml file to create a Service.
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    tier: application
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    run: nginx
  type: NodePort
```

Here is an example of Service.

* IP`10.96.17.77` is ClusterIP(VIP) of the service
* Port `<unset>  80/TCP` is the port on Pod that service listening within the cluster.
* TargetPort `8080/TCP` is the port on the container that the service should direct traffic to.
* NodePort `<unset>  31893/TCP` is the port that can be accessed outside. Default range is `30000~32767`. The port is exposed across **all** nodes in cluster.
* Endpoints show the list of Pods matched the service labels. 

```
Name:                     nginx-deployment
Namespace:                jh-namespace
Labels:                   tier=application
Annotations:              <none>
Selector:                 run=nginx
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.17.77
IPs:                      10.96.17.77
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31893/TCP
Endpoints:                10.244.1.177:8080,10.244.1.178:8080,10.244.1.179:8080 + 7 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```




Service `kube-dns` beyond Deployment `coredns` provides cluster DNS service in Kubernetes cluster. 

Service registration:

* Kubernetes uses cluster DNS as service registration.
* Registration is Service based, not Pod based.
* Cluster DNS (CoreDNS) is monitoring and discvering new service actively.
* Service Name, IP, Port will be registered.


Procedure of Service registration.

* POST new Service to API Server.
* Assign ClusterIP to the new Service.
* Save new Service configuration info to etcd.
* Create endpoints with related Pod IPs associated with the new Service.
* Explore the new Service by ClusterDNS.
* Create DNS info.
* kube-proxy fetch Service configration info.
* Create IPSV rule.



Procedure of Service discovery.

* Request DNS name resolution for a Service name.
* Receive ClusterIP.
* Traffic access to ClusterIP.
* No router. Forward request to Pod's default gateway.
* Forward request to node.
* No router. Forward request to Node's default gateway.
* Proceed the request by Node kernel.
* Trap the request by IPSV rule.
* Put destination Pod's IP into the request's destination IP. 
* The request arrives destination Pod.




FQDN format: `<object-name>.<namespace>.svc.cluster.local`. We call `<object-name>` as unqualified name, or short name.
Namespaces can segregate the cluster's address space. At the same time, it can also be used to implement access control and resource quotas.

Get DNS configuration in a Pod. 
The IP of nameserver is same with ClusterIP of kube-dns Service, which is well-known IP for request of DNS or service discovery.
```
root@cka001:/etc# kubectl get service kube-dns -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   7d7h


root@cka001:~# kubectl exec -it nginx-5f5496dc9-bv5dx -- /bin/bash
root@nginx-5f5496dc9-bv5dx:/# cat /etc/resolv.conf
search jh-namespace.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```


Get information of `kube-dns`.
```
root@cka001:~# kubectl describe service kube-dns -n kube-system
Name:              kube-dns
Namespace:         kube-system
Labels:            k8s-app=kube-dns
                   kubernetes.io/cluster-service=true
                   kubernetes.io/name=CoreDNS
Annotations:       prometheus.io/port: 9153
                   prometheus.io/scrape: true
Selector:          k8s-app=kube-dns
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.0.10
IPs:               10.96.0.10
Port:              dns  53/UDP
TargetPort:        53/UDP
Endpoints:         10.244.0.2:53,10.244.0.3:53
Port:              dns-tcp  53/TCP
TargetPort:        53/TCP
Endpoints:         10.244.0.2:53,10.244.0.3:53
Port:              metrics  9153/TCP
TargetPort:        9153/TCP
Endpoints:         10.244.0.2:9153,10.244.0.3:9153
Session Affinity:  None
Events:            <none>
```









### Endpoints 

Endpoints is a collection of endpoints that implement the actual service.

When a service is created, it associates with a Endpoint object, `kubectl get endpoints <service_name>`.

A list of matched Pod by service label is maintained as Endpoint object, add new matched Pods and remove not matched Pods.








## Config and Storage Resources

### Volumes

#### emptyDir

An `emptyDir` volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running on that node. 

The `emptyDir` volume is initially empty. 

All containers in the Pod can read and write the same files in the `emptyDir` volume, though that volume can be mounted at the same or different paths in each container. 

When a Pod is removed from a node for any reason, the data in the `emptyDir` is deleted permanently.

A container crashing does not remove a Pod from a node. The data in an `emptyDir` volume is safe across container crashes.


Usage:

* scratch space, such as for a disk-based merge sort
* checkpointing a long computation for recovery from crashes
* holding files that a content-manager container fetches while a webserver container serves the data






#### hostPath

A `hostPath` volume mounts a file or directory from the host node's filesystem into your Pod. 
This is not something that most Pods will need, but it offers a powerful escape hatch for some applications.

`hostPath` volumes present many security risks, and it is a best practice to avoid the use of HostPaths when possible. 
When a HostPath volume MUST be used, it should be scoped to only the required file or directory, and mounted as ReadOnly.

If restricting HostPath access to specific directories through AdmissionPolicy, volumeMounts MUST be required to use readOnly mounts for the policy to be effective.


Usage: 

* Running together with DaemonSet, e.g., EFK Fluentd mount log directory of local host in order to collect host log information.
* Running on a specific node by using `hostPath` volumne, which can get high performance disk I/O.
* Running a container that needs access to Docker internals; use a hostPath of `/var/lib/docker`.
* Running cAdvisor in a container; use a hostPath of `/sys`.
* Allowing a Pod to specify whether a given hostPath should exist prior to the Pod running, whether it should be created, and what it should exist as.






### Storage Class
Procedure of StorageClass deployment and implementation:

* Create Kubernetes cluster and backend storage.
* Make sure the provisioner/plugin is ready in Kubernetes.
* Create a StorageClass object to link to backend storage. The StorageClass will create related PV automatically.
* Create a PVC object to link to the StorageClass we created.
* Deploy a Pod and use the PVC volume.



### PV

PV Recycle Policy.

* Retain.
* Delete.
* Recycle. 


PV in-tree type:

* hostPath
* local
* NFS
* CSI


### Access Modes
`spec.accessModes` defines mount option of a PV:

* ReadWriteOnce(RWO). A PV can be mounted only to a PVC with read/write mode, like block device.
* ReadWriteMany(RWM). A PV can be mounted to more than one PVC with read/write mode, like NFS.
* ReadOnlyMany(ROM). A PV can be mounted to more than one PVC with read only mode.
* ReadWriteOncePod (RWOP). Only support CSI type PV, can be mounted by single Pod.

A PV can only be set with one option. Pod mount PVC, not PV.












