# CKA自学笔记5:Kubernetes基本概念散记

## Kubernetes基本概念

### Kubernetes组件

一个Kubernetes集群由代表控制平面（control plane）的组件和一组称为节点（nodes）的机器组成。

![The components of a Kubernetes cluster](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

**Kubernetes组件**: 

* 控制平面组件 Control Plane Components
  * kube-apiserver: 
    * 查询和操作 Kubernetes 中对象的状态。
    * 充当所有资源之间的通信中心（communication hub）。
    * 提供集群安全身份验证、授权和角色分配。
    * 是唯一能连接到 etcd 的组件。
  * etcd: 
    * 所有 Kubernetes 对象都存储在 `etcd` 中。
    * Kubernetes 对象是 Kubernetes 系统中的持久实体(entities)，用于表示集群的状态。
  * kube-scheduler: 
    * 监视没有分配节点的新创建的 Pod，并为它们选择一个节点来运行。
  * kube-controller-manager: 
    * 运行控制器进程。
    * *Node controller*: 负责警示和响应节点的故障。
    * *Job controller*: 监视表示一次性任务的 Job 对象，然后创建 Pod 来完成这些任务。
    * *Endpoints controller*: 填充 Endpoints 对象（即将 Service 和 Pod 连接起来）。
    * *Service Account & Token controllers*: 为新命名空间创建默认帐户和 API 访问令牌。
  * cloud-controller-manager: 
    * 嵌入云特定的控制逻辑，仅运行特定于我们选择的云提供商的控制器，无需自己的基础设施和学习环境。
    * *Node controller*: 用于检查云提供商，以确定节点在在它停止响应后是否已在云中被删除。
    * *Route controller*: 用于在底层云基础架构中设置路由。
    * *Service controller*: 用于创建、更新和删除云提供商负载均衡器。
* 节点组件 Node Components
  * kubelet: 
    * 在集群中每个节点上运行的代理。 
    * 管理节点。它确保 Pod 中运行容器。`kubelet` 向 APIServer 注册和更新节点信息，APIServer 将它们存储到 `etcd` 中。
    * 管理 Pod。通过 APIServer 监视 Pod，并对 Pod 或 Pod 中的容器采取行动。
    * 在容器级别进行健康检查。
  * kube-proxy: 
    * 是在集群中每个节点上运行的网络代理。
      * iptables
      * ipvs
    * 维护节点上的网络规则。
  * 容器运行时Container runtime：
    * 负责运行容器的软件。
* 插件Addons
  * DNS: 是 DNS 服务器，是所有 Kubernetes 集群所必需的。
  * Web UI（仪表盘）：用于 Kubernetes 集群的基于 Web 的用户界面。
  * 容器资源监控：记录有关集中式数据库中容器的通用时间序列度量。
  * Cluster-level Logging：负责将容器日志保存到具有搜索/浏览接口的中央日志存储中。

可扩展性：

- 水平扩展（Scaling out）通过添加更多的服务器到架构中，将工作负载分散到更多的机器上。
- 垂直扩展（Scaling up）通过添加更多的硬盘和内存来增加物理服务器的计算能力。

### Kubernetes API

REST API是Kubernetes的基本框架。所有组件之间的操作和通信，以及外部用户命令都是由API服务器处理的REST API调用。因此，Kubernetes平台中的所有内容都被视为API对象（API object），并在API中有相应的条目。

Kubernetes控制平面的核心是API服务器。

- CRI：容器运行时接口
- CNI：容器网络接口
- CSI：容器存储接口

API服务器公开了一个HTTP API，允许最终用户、集群的不同部分和外部组件彼此通信。

Kubernetes API允许我们查询和操作Kubernetes中API对象的状态（例如：Pod、Namespace、ConfigMap和Event）。

Kubernetes API：

- OpenAPI规范
  - OpenAPI V2
  - OpenAPI V3
- 持久性。Kubernetes通过将对象的序列化状态写入etcd来存储它们。
- API组和版本控制。版本控制是在API级别进行的。API资源通过它们的API组、资源类型、命名空间（用于命名空间资源）和名称进行区分。
  - API更改
- API扩展



#### API Version

API版本和软件版本间存在间接关系。API和发布版本计划描述了API版本和软件版本之间的关系。不同的API版本表示不同的稳定性和支持级别。

以下是每个级别的摘要：

- Alpha：
  - 版本名称包含alpha（例如，v1alpha1）。
  - 软件可能包含错误。启用功能可能会暴露错误。某些功能可能默认禁用。
  - 对于某些功能的支持可以随时取消，而不会提前通知。
  - API可能会在以后的软件发布中以不兼容的方式更改，而不会提前通知。
  - 由于错误风险增加和长期支持不足，建议仅在短暂的测试集群中使用该软件。
- Beta：
  - 版本名称包含beta（例如，v2beta3）。
  - 软件经过充分测试。启用功能被认为是安全的。某些功能默认启用。
  - 对于某些功能的支持不会取消，但细节可能会更改。
  - 对象的模式和/或语义可能会在后续的Beta或稳定版发布中以不兼容的方式更改。当发生这种情况时，将提供迁移说明。模式更改可能需要删除、编辑和重新创建API对象。编辑过程可能不简单。迁移可能需要停机，以便依赖于该功能的应用程序。
  - 不建议将该软件用于生产用途。后续的发布可能会引入不兼容的更改。如果您有多个可以独立升级的集群，则可以放宽此限制。
    注意：请尝试beta功能并提供反馈。功能退出beta后，可能不实际再进行更改。
- 稳定版：
  - 版本名称为vX，其中X是整数。
  - 功能的稳定版本出现在发布的软件中的许多后续版本中。

读取当前API的版本命令：

```bash
kubectl api-resources
```

#### API Group

[API组（API groups）](https://git.k8s.io/design-proposals-archive/api-machinery/api-group.md)使扩展Kubernetes API更加容易。API组在REST路径和序列化对象的apiVersion字段中指定。

Kubernetes有几个API组：

- 核心组（也称为遗留legacy）位于REST路径 `/api/v1`。
  - 核心组不作为apiVersion字段的一部分指定，例如 apiVersion: v1。
- 命名组位于REST路径 `/apis/$GROUP_NAME/$VERSION`，并使用 apiVersion: `$GROUP_NAME/$VERSION`（例如 apiVersion: batch/v1）。



### Kubernetes对象

#### 对象概述

对象规范（Object Spec）：

- 提供了一个描述所创建资源的特性的说明：*其期望的状态*。

对象状态（Object Status）：

- 描述了对象的当前状态。



比如，Deployment是一个可以代表集群上运行的应用程序的对象。

```yaml
apiVersion: apps/v1  # 当前用来创建对象的API版本
kind: Deployment     # 创建对象的类型
metadata:            # 用来区分对象的元数据，比如：名称，UID，命名空间等
  name: nginx-deployment
spec:                # 期望所创建对象的状态
  selector:
    matchLabels:
      app: nginx
  replicas: 2        # 告诉Deployment基于下面的模板template创建2个Pods
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

#### 对象管理

`kubectl` 命令行工具支持多种不同的方式来创建和管理 Kubernetes 对象。详细信息请阅读 [Kubectl book](https://kubectl.docs.kubernetes.io/)。

一个 Kubernetes 对象应该仅使用一种技术进行管理。混合使用不同的技术来管理同一个对象会导致非预期的结果。

三种管理技术:

- 命令式命令
  - 直接在集群中操作实时对象。
  - `kubectl create deployment nginx --image nginx`
- 命令式对象配置
  - `kubectl create -f nginx.yaml`
  - `kubectl delete -f nginx.yaml -f redis.yaml`
  - `kubectl replace -f nginx.yaml`
- 声明式对象配置
  - `kubectl diff -f configs/`
  - `kubectl apply -f configs/`



#### 对象名称和ID

集群中的每个对象都有一个在该资源类型中唯一的名称。

- DNS 子域名
- 标签名称
- 路径段名称

每个 Kubernetes 对象还有一个 UID，在整个集群中是唯一的。

#### 命名空间

在Kubernetes中，命名空间提供了一种在单个集群内隔离资源组的机制。

资源的名称需要在命名空间内是唯一的，但不需要跨命名空间唯一。

基于命名空间的范围仅适用于命名空间对象（例如部署，服务等），而不适用于集群范围的对象（例如StorageClass，节点，持久卷等）。

并非所有对象都位于命名空间中。

Kubernetes从四个初始命名空间开始：

- `default` 用于没有其他命名空间的对象的默认命名空间
- `kube-system` Kubernetes系统创建的对象的命名空间
- `kube-public` 该命名空间是自动创建的，并可由所有用户（包括未经身份验证的用户）读取。此命名空间大多保留供集群使用，以防一些资源应在整个集群范围内公开和可读。此命名空间的公共方面只是一种约定，而不是要求。
- `kube-node-lease` 此命名空间保存与每个节点关联的租赁对象。节点租赁允许kubelet发送心跳，以便控制平面可以检测到节点故障。

查看命名空间：

- `kubectl get namespace`

为请求设置命名空间

- `kubectl run nginx --image=nginx --namespace=<插入命名空间名称>`
- `kubectl get pods --namespace=<插入命名空间名称>`





#### 标签和选择器

标签是附加到对象（例如 Pod）的键/值对。有效的标签键有两个部分：可选的前缀和名称，由斜杠（`/`）分隔。

标签旨在用于指定对用户有意义和相关的对象识别属性。

标签可用于组织和选择对象子集。标签可以在创建对象时附加，随后在任何时候添加和修改。每个对象可以定义一组键/值标签，每个键必须对于给定对象是唯一的。

标签的示例：

```yaml
"metadata": {
    "labels": {
        "key1" : "value1",
        "key2" : "value2"
    }
}
```

与名称和 UID 不同，标签不提供唯一性。通常情况下，我们期望许多对象带有相同的标签。

目前 API 支持两种类型的选择器：

- 基于等式的选择器，例如：`environment = production`、`tier != frontend`
- 基于集合的选择器，例如：`environment in (production, qa)`、`tier notin (frontend, backend)`

例如：

```bash
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production),tier in (frontend)'
kubectl get pods -l 'environment in (production, qa)'
kubectl get pods -l 'environment,environment notin (frontend)'
```

#### 注释Annotations

使用 Kubernetes 注释（Annotations）将任意非标识元数据附加到对象上。 工具和库等客户端可以检索此元数据。

使用标签或注释将元数据附加到 Kubernetes 对象上。

- 标签可用于选择对象并查找满足某些条件的对象集合。
- 注释不用于标识和选择对象。

注释与标签类似，都是键/值映射。 映射中的键和值必须是字符串。

例如：

```yaml
"metadata": {
    "annotations": {
      "key1" : "value1",
      "key2" : "value2"
    }
}
```

合法的注释键具有两个部分：可选的前缀和名称，由斜杠 (`/`) 分隔。

#### 字段选择器

字段选择器（field selectors）可以根据一个或多个资源字段的值选择Kubernetes资源。

下面是一些使用字段选择器进行查询筛选的例子：

```yaml
metadata.name=my-service
metadata.namespace!=default
status.phase=Pending
```

This kubectl command selects all Pods for which the value of the status.phase field is Running:
`kubectl get pods --field-selector status.phase=Running`

Supported field selectors vary by Kubernetes resource type. All resource types support the `metadata.name` and `metadata.namespace` fields. 

Use the `=`, `==`, and `!=` operators with field selectors (`=` and `==` mean the same thing). 

下面 kubectl 命令选择所有状态(phase)字段值为 Running 的 Pod： 

```bash
kubectl get pods --field-selector status.phase=Running
```

支持的字段选择器因 Kubernetes 资源类型而异。所有资源类型都支持 `metadata.name` 和 `metadata.namespace` 字段。

在字段选择器中使用 `=`, `==`, 和 `!=` 运算符(`=` 和 `==` 表示相同的意思)。

例如：

```bash
kubectl get ingress --field-selector foo.bar=baz

kubectl get services --all-namespaces --field-selector metadata.namespace!=default

kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always

kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```

Finalizers是*命名空间键*，告诉Kubernetes在满足特定条件之前等待，然后再完全删除标记为*删除*的资源。 Finalizer警告控制器controller清理已删除对象所拥有的资源。

通常因为某种目的为资源添加Finalizers，强制删除它们可能会导致集群中出现问题。

与标签类似，*所有者引用*（Owner references）描述了Kubernetes中对象之间的关系，但用于不同的目的。

Kubernetes使用所有者引用（而不是标签）来确定集群中哪些Pod需要清理。

当Kubernetes识别到目标删除的资源上有所有者引用时，它会处理Finalizer。

#### 所有者和依赖关系

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
