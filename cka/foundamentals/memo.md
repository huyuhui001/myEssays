# CKA自学笔记5:Kubernetes随笔

## 摘要

边练习边记录的内容，不是全面系统的，包括下面主要内容：

- Kubernetes基本概念
  - 组件
  - API
  - 对象
  - 资源
- 工作负载资源
  - Pod
  - Deployment
  - ReplicaSet
  - StatefulSet
  - DaemonSet
  - Job
  - CronJob
- 服务资源
  - Service
  - Endpoints
- 配置和存储资源
  - 卷
  - Storage Class
  - PV
  - Access Modes

## Kubernetes基本概念

### Kubernetes组件

一个Kubernetes集群由代表控制平面（control plane）的组件和一组称为节点（nodes）的机器组成。

![The components of a Kubernetes cluster](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

Kubernetes组件: 

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
  - 对象的模式和/或语义可能会在后续的Beta或稳定版发布中以不兼容的方式更改。当发生这种情况时，将提供迁移说明。模式更改可能需要删除、编辑和重新创建  API对象。编辑过程可能不简单。迁移可能需要停机，以便依赖于该功能的应用程序。
  - 不建议将该软件用于生产用途。后续的发布可能会引入不兼容的更改。如果您有多个可以独立升级的集群，则可以放宽此限制。
  - 注意：请尝试beta功能并提供反馈。功能退出beta后，可能不实际再进行更改。
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

在 Kubernetes 中，一些对象拥有其他对象。例如，ReplicaSet 是一组 Pod 的所有者。这些被拥有的对象是其所有者的从属对象。

从属对象具有一个 `metadata.ownerReferences` 字段，该字段引用其所有者对象。

有效的所有者引用包括对象名称和与从属对象相同的命名空间中的 UID。

从属对象还具有一个 `ownerReferences.blockOwnerDeletion` 字段，该字段具有布尔值，控制特定的从属对象是否可以阻止垃圾回收删除其所有者对象。

### 资源

Kubernetes资源和“意向记录”都以API对象的形式存储，并通过对API的RESTful调用进行修改。

API允许以声明性方式管理配置。

用户可以直接与Kubernetes
 API交互，也可以通过像kubectl这样的工具进行交互。

核心Kubernetes API具有灵活性，也可以扩展以支持自定义资源。

* 工作负载资源（Workload Resources）
  - *Pod*。Pod 是可以在主机上运行的容器集合。
  - *PodTemplate*。PodTemplate 描述了预定义 pod 的副本模板。
  - *ReplicationController*。ReplicationController 表示一个复制控制器的配置。
  - *ReplicaSet*。ReplicaSet 确保在任何给定时间有指定数量的 pod 副本正在运行。
  - *Deployment*。Deployment 使 Pod 和 ReplicaSet 的声明性更新成为可能。
  - *StatefulSet*。StatefulSet 表示具有一致标识的 pod 集合。
  - *ControllerRevision*。ControllerRevision 实现了状态数据的不可变快照。
  - *DaemonSet*。DaemonSet 表示一个守护进程集的配置。
  - *Job*。Job 表示单个 job 的配置。
  - *CronJob*。CronJob 表示单个 cron job 的配置。
  - *HorizontalPodAutoscaler*。HorizontalPodAutoscaler 表示水平 pod 自动缩放器的配置。
  - *HorizontalPodAutoscaler v2beta2*。HorizontalPodAutoscaler 是水平 pod 自动缩放器的配置，根据指定的指标自动管理实现比例子资源的任何资源的副本计数。
  - *PriorityClass*。PriorityClass 定义了从优先级类名称到优先级整数值的映射。
* 服务资源（Service Resources）
  - *Service*. Service 是对软件服务（例如mysql）的命名抽象，由代理监听的本地端口（例如3306）和确定哪些Pod将回答通过代理发送的请求的选择器组成。
  - *Endpoints*. Endpoints 是实现实际服务的一组终结点。
  - *EndpointSlice*. EndpointSlice 表示实现服务的终结点的子集。
  - *Ingress*. Ingress 是一组规则，允许入站连接到达由后端定义的终结点。
  - *IngressClass*. IngressClass 表示 Ingress 的类，由 Ingress Spec 引用。
* 配置和存储资源（Config and Storage Resources）
  - *ConfigMap*。ConfigMap保存容器需要使用的配置数据。
  - *Secret*。Secret保存特定类型的机密数据。
  - *Volume*。Volume表示Pod中的命名卷，可以被Pod中的任何容器访问。
  - *PersistentVolumeClaim*。PersistentVolumeClaim是用户对持久卷的请求和声明。
  - *PersistentVolume*。PersistentVolume（PV）是由管理员提供的存储资源。
  - *StorageClass*。StorageClass描述可动态分配PersistentVolumes的存储类别的参数。
  - *VolumeAttachment*。VolumeAttachment记录将指定的卷附加到/从指定节点中分离的意图。
  - *CSIDriver*。CSIDriver记录集群上部署的容器存储接口（CSI）卷驱动程序的信息。
  - *CSINode*。CSINode保存有关节点上安装的所有CSI驱动程序的信息。
  - *CSIStorageCapacity*。CSIStorageCapacity存储一个CSI GetCapacity调用的结果。
* 认证资源（Authentication Resources）
  - ServiceAccount*。ServiceAccount和下面的信息绑定在一起：
    - 一个可被用户和周边系统理解的名称，用于身份识别
    - 可进行身份验证和授权的主体
    - 一组密钥。
  - TokenRequest*。TokenRequest为给定的ServiceAccount请求一个令牌。
  - TokenReview*。TokenReview尝试对已知用户的令牌进行身份验证。
  - CertificateSigningRequest*。CertificateSigningRequest对象提供了一种通过提交证书签名请求并异步批准和发放来获取x509证书的机制。
* 授权资源（Authorization Resources）
  - LocalSubjectAccessReview*。LocalSubjectAccessReview检查一个用户或组在给定命名空间中是否能执行某个操作。
  - SelfSubjectAccessReview*。SelfSubjectAccessReview检查当前用户是否能执行某个操作。
  - SelfSubjectRulesReview*。SelfSubjectRulesReview枚举当前用户在一个命名空间内可以执行的操作集合。
  - SubjectAccessReview*。SubjectAccessReview检查一个用户或组是否能执行某个操作。
  - ClusterRole*。ClusterRole是一个集群级别的PolicyRules逻辑分组，可以被RoleBinding或ClusterRoleBinding引用为一个单元。
  - ClusterRoleBinding*。ClusterRoleBinding引用一个ClusterRole，但不包含它。
  - Role*。Role是一个命名空间级别的PolicyRules逻辑分组，可以被RoleBinding引用为一个单元。
  - RoleBinding*。RoleBinding引用一个Role，但不包含它。
* 策略资源（Policy Resources）
  - LimitRange*。LimitRange为命名空间中每种资源设置资源使用限制。
  - ResourceQuota*。ResourceQuota设置每个命名空间强制执行的总配额限制。
  - NetworkPolicy*。NetworkPolicy描述了一组Pod允许的网络流量。
  - PodDisruptionBudget*。PodDisruptionBudget是一个对象，用于定义对一组Pod可能造成的最大中断。
  - PodSecurityPolicy v1beta1*。PodSecurityPolicy控制对可能影响将应用于Pod和容器的安全上下文的请求的能力。
* 扩展资源（Extend Resources）
  - CustomResourceDefinition*。CustomResourceDefinition表示应在API服务器上公开的资源。
  - MutatingWebhookConfiguration*。MutatingWebhookConfiguration描述接受或拒绝并可能更改对象的准入Webhook的配置。
  - ValidatingWebhookConfiguration*。ValidatingWebhookConfiguration描述接受或拒绝对象但不更改对象的准入Webhook的配置。
* 集群资源（Cluster Resources）
  - Node*。Node是Kubernetes中的工作节点。
  - Namespace*。Namespace为名称提供了作用域。
  - Event*。Event是对集群中某个位置发生事件的报告。
  - APIService*。APIService表示特定GroupVersion的服务器。
  - Lease*。Lease定义了租赁的概念。
  - RuntimeClass*。RuntimeClass定义了集群中支持的容器运行时类。
  - FlowSchema v1beta2*。FlowSchema定义了一组流程的架构。
  - PriorityLevelConfiguration v1beta2*。PriorityLevelConfiguration表示优先级级别的配置。
  - Binding*。Binding将一个对象绑定到另一个对象；例如，调度程序将Pod绑定到节点上。
  - ComponentStatus*。ComponentStatus（和ComponentStatusList）保存集群验证信息。

使用命令 `kube api-resources` 获取支持的API资源。

使用命令 `kubectl explain RESOURCE [options]` 描述与每个支持的API资源相关联的字段。这些字段可以通过简单的JSONPath标识符进行识别：

```bash
kubectl explain binding
kubectl explain binding.metadata
kubectl explain binding.metadata.name
```

## 工作负载资源

### Pods

Pod是Kubernetes中可创建和管理的最小部署计算单位。

Pod是一个包含一个或多个容器、共享存储和网络资源以及如何运行容器的规范的组。

Pod的内容始终共同定位和共同安排，并在共享环境中运行。

Pod模拟了一个特定于应用程序的“逻辑主机”：它包含一个或多个相对紧密耦合的应用程序容器。

在非云环境中，同一物理或虚拟机上执行的应用程序类似于在同一逻辑主机上执行的云应用程序。

Pod的共享环境是一组Linux命名空间、cgroups和可能的其他隔离要素 - 这些要素与隔离Docker容器的方式相同。

在Docker概念方面，Pod类似于具有共享命名空间和共享文件系统卷的一组Docker容器。

通常情况下，甚至是单例Pod，我们都不需要直接创建Pod，而是使用工作负载资源，例如*Deployment*或*Job*来创建它们。如果Pod需要跟踪状态，则可以使用StatefulSet资源。

Kubernetes集群中的Pod有两种主要用法：

- 运行单个容器的Pod。
- 运行需要共同工作的多个容器的Pod。

“每个Pod一个容器”的模型是最常见的Kubernetes用例；在这种情况下可以将Pod视为单个容器的包装器；Kubernetes管理Pod而不是直接管理容器。

一个Pod可以封装由多个共同定位、紧密耦合且需要共享资源的容器组成的应用程序。

这些共同定位的容器形成一个单一的服务整体单元 - 例如，一个容器向公众提供存储在共享卷中的数据，而另一个独立的Sidecar容器刷新或更新这些文件。Pod将这些容器、存储资源和短暂的网络标识包装在一起，作为一个单独的单位。

在单个Pod中分组多个共同定位和共同管理的容器是相对高级的用例。应该*仅在*容器紧密耦合的特定情况下使用此模式。

每个Pod都旨在运行给定应用程序的单个实例。如果我们想水平扩展您的应用程序（通过运行更多实例提供更多的总资源），则应该使用多个Pod，每个实例一个Pod。在Kubernetes中，这通常称为*复制*。复制的Pod通常作为工作负载资源及其控制器的一组创建和管理。

Pod本地提供两种共享资源以供其组成容器使用：*[网络](https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking)*和*[存储](https://kubernetes.io/docs/concepts/workloads/pods/#pod-storage)*。

一个Pod可以指定一组共享的存储卷。Pod中的所有容器都可以访问这些共享卷，使这些容器可以共享数据。

每个Pod为每个地址族分配一个唯一的IP地址。
在一个Pod内，容器共享一个IP地址和端口空间，并可以通过“localhost”找到彼此。想要与运行在不同Pod中的容器交互的容器可以使用IP网络进行通信。

当创建一个Pod时，新的Pod被调度在集群中的一个节点上运行。Pod保留在该节点上，直到Pod执行完毕、Pod对象被删除、Pod因缺乏资源而被驱逐或节点发生故障。

在Pod中重新启动一个容器不应与重新启动一个Pod混淆。Pod不是一个进程，而是一个运行容器的环境。Pod会一直保留，直到被删除为止。

您可以使用工作负载资源（例如Deployment、StatefulSet、DaemonSet）为自己创建和管理多个Pod。资源的控制器处理复制、滚动和在Pod失败时的自动恢复。![Pod with multiple containers](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)

#### 初始化容器

一些Pod还有初始化容器（Init containers）和应用容器（app containers）。初始化容器在应用容器启动之前运行并完成。

我们可以在Pod规范中指定初始化容器，同时也可以在容器数组中描述应用容器。

#### 静态Pod

静态Pod是直接由特定节点上的kubelet守护程序管理的，API服务器不会观察它们。

静态Pod始终绑定到特定节点上的一个Kubelet。

静态Pod的主要用途是运行自托管控制面板：换句话说，使用kubelet监督各个控制面板组件。

kubelet会自动尝试为每个静态Pod在Kubernetes API服务器上创建一个镜像Pod。这意味着在节点上运行的Pod在API服务器上可见，但无法从那里控制。

#### 容器探针

探针是 kubelet 定期对容器执行的一种诊断。

为执行诊断，kubelet 要么在容器内执行代码，要么发起网络请求。

使用探针有四种不同的检查容器方式。每个探针必须恰好定义这四种机制中的一种：

- *exec*。如果命令以状态代码 0 退出，则将诊断视为成功。
- *grpc*。如果响应的状态为 SERVING，则将诊断视为成功。
- *httpGet*。如果响应的状态代码大于或等于 200 且小于 400，则将诊断视为成功。
- *tcpSocket*。如果端口开放，则将诊断视为成功。

每个探针有三种结果：

- 成功
- 失败
- 未知

探针类型：

- *livenessProbe*。指示容器是否正在运行。
- *readinessProbe*。指示容器是否准备好响应请求。
- *startupProbe*。指示容器内的应用程序是否启动。

### Deployment

### ReplicaSet

ReplicaSet的目的是在任何时候维护一组稳定的副本Pod。因此，它通常用于保证指定数量的相同Pod的可用性。

我们一般不需要直接操纵ReplicaSet对象：使用Deployment，然后在spec部分中定义您的应用程序。

可以通过设置`replicaset.spec.replicas`来指定应同时运行多少个Pod。 ReplicaSet将创建/删除其Pod以匹配此数字。
如果不指定`replicaset.spec.replicas`，则默认值为`1`。

### StatefulSet

StatefulSet 特点（又称固定标识）：

- Pod 的名称在创建后不可变。
- DNS 主机名在创建后不可变。
- 挂载的卷在创建后不可变。

StatefulSet 的固定标识在失败、扩展和其他操作后不会改变。

StatefulSet 的命名约定为：`<StatefulSetName>-<Integer>`。

StatefulSet 可以自行进行扩展，但是 Deployment 需要依靠 ReplicaSet 进行扩展。

建议：先将 StatefulSet 减少到 0，而不是直接删除它。

*headless* Service 和 *governing* Service：

- Headless Service 是一个普通的 Kubernetes Service 对象，其 `spec.clusterIP` 被设置为 `None`。
- 当 StatefulSet 的 `spec.ServiceName` 设置为 headless Service 名称时，StatefulSet 现在是一个 governing Service。

创建 StatefulSet 的一般过程：

- 创建 StorageClass。
- 创建 Headless Service。
- 基于上述两个创建 StatefulSet。

### DaemonSet

DaemonSet保证所有（或部分）节点运行Pod的副本。随着节点从集群中删除，这些Pod将被垃圾回收。

删除DaemonSet将清理它创建的Pod。

一些典型DaemonSet的用途包括：

- 在每个节点上运行集群存储守护程序。
- 在每个节点上运行日志收集守护程序。
- 在每个节点上运行节点监视守护程序。

在简单的情况下，每种类型的守护程序将使用覆盖所有节点的一个DaemonSet。

更复杂的设置可能会为单个守护程序使用多个DaemonSet，但使用不同的标志和/或不同的内存和CPU请求来支持不同的硬件类型。

DaemonSet控制器调和过程同时检查现有节点和新创建的节点。

默认情况下，Kubernetes调度程序忽略由DamonSet创建的Pod，并允许它们存在于节点上，直到关闭节点本身。

在选择节点上运行Pod：

- 如果您指定`daemonset.spec.template.spec.nodeSelector`，那么DaemonSet控制器将在与该节点选择器匹配的节点上创建Pod。
- 如果您指定`daemonset.spec.template.spec.affinity`，那么DaemonSet控制器将在与该节点亲和力匹配的节点上创建Pod。
- 如果两者都不指定，则DaemonSet控制器将在所有节点上创建Pod。

在`kubectl explain daemonset.spec`中没有`replicas`字段与`kubectl explain deployment.spec.replicas`相对应。当创建一个DaemonSet时，每个节点将运行*一个* DaemonSet Pod。

对于服务，通常是无状态的，一般不关心节点在哪里运行，更关心Pod副本的数量，并且我们可以将这些副本/replicas缩放。在这里，滚动更新也将是一个优点。

当Pod的一个副本必须在某个指定节点上运行时，我们将使用`DaemonSet`。我们的守护进程Pod还需要在我们的其他Pod之前启动。

DaemonSet是用于后台服务的简单可扩展性策略。当更多的合适的节点添加到集群时，后台服务将扩展。当节点被删除时，它将自动缩小。

### Job

### CronJob

## 服务资源

### Service

Service是软件服务（例如mysql）的命名抽象，由代理监听的本地端口（例如3306）和确定哪些Pod将回答通过代理发送的请求的选择器组成。

一个Service的目标Pod集合通常由一个选择器（标签选择器）来确定。

Service资源的类型包括：

- ClusterIP Service（默认）：可靠的IP、DNS和端口。仅限内部访问。
- NodePort Service：向外部提供访问。
- LoadBalancer：基于NodePort，并与云供应商提供的负载平衡集成（例如AWS、GCP等）。
- ExternalName：访问将被转发到外部服务。

下面是一个创建简单Service的yaml文件：

```yaml
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

下面是一个Service的例子：

- IP `10.96.17.77` 是该服务的 ClusterIP(VIP)。
- 端口 `<unset> 80/TCP` 是 Pod 在集群内监听的端口。
- TargetPort `8080/TCP` 是容器内服务应该定向流量到达的端口。
- NodePort `<unset> 31893/TCP` 是可以从外部访问的端口。默认范围是 `30000~32767`。该端口会在整个集群的所有节点上暴露。
- Endpoints 显示了匹配服务标签的 Pod 列表。

```yaml
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

在 Kubernetes 集群中，基于Deployment `coredns` 的Service `kube-dns`提供了集群 DNS 服务。

服务注册：

- Kubernetes 使用集群 DNS 作为服务注册。
- 注册是基于 Service 而非 Pod 的。
- 集群 DNS（CoreDNS）主动监视和发现新服务。
- Service 名称、IP、端口将被注册。

Service 注册的过程如下：

- 将新的 Service POST 到 API Server。
- 为新的 Service 分配 ClusterIP。
- 将新的 Service 配置信息保存到 etcd 中。
- 创建与新 Service 关联的带有相关 Pod IP 的 endpoints。
- 通过 ClusterDNS 探索新的 Service。
- 创建 DNS 信息。
- kube-proxy 获取 Service 配置信息。
- 创建 IPSV 规则。

Service 发现的过程。

- 请求一个 Service 名称的 DNS 名称解析。
- 收到 ClusterIP。
- 访问 ClusterIP。
- 没有路由器。将请求转发到 Pod 的默认网关。
- 将请求转发到节点。
- 没有路由器。将请求转发到节点的默认网关。
- 节点内核继续处理请求。
- 使用 IPSV 规则捕获请求。
- 将目标 Pod 的 IP 放入请求的目标 IP 中。
- 请求到达目标 Pod。

FQDN格式为：`<object-name>.<namespace>.svc.cluster.local`。我们称`<object-name>`为非限定名称或简短名称。
命名空间可以隔离集群的地址空间。同时，它还可以用于实现访问控制和资源配额。

获取Pod中的DNS配置。
nameserver的IP与kube-dns服务的ClusterIP相同，这是用于DNS请求或服务发现请求的众所周知的IP。

```bash
$ kubectl get service kube-dns -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   7d7h


$ kubectl exec -it nginx-5f5496dc9-bv5dx -- /bin/bash
root@nginx-5f5496dc9-bv5dx:/# cat /etc/resolv.conf
search jh-namespace.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

读取 `kube-dns`信息：

```bash
$ kubectl describe service kube-dns -n kube-system
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

Endpoints是一组实现实际服务的端点集合。

当创建服务时，它会与一个Endpoint对象相关联，可以使用命令 `kubectl get endpoints <service_name>` 获取。

匹配服务标签的Pod列表维护为Endpoint对象，添加新的匹配Pod并删除不匹配的Pod。

## 配置和存储资源

### 卷

#### emptyDir

`emptyDir`卷是在Pod分配到节点时首先创建的，并且只要该Pod在该节点上运行，它就会存在。

`emptyDir`卷最初为空。

Pod中的所有容器都可以读取和写入`emptyDir`卷中的相同文件，尽管该卷可以在每个容器中以相同或不同的路径挂载。

当由于任何原因从节点中删除Pod时，`emptyDir`中的数据将永久删除。

容器崩溃不会将Pod从节点中删除。 `emptyDir`卷中的数据可以在容器崩溃时安全保留。

用途：

- 临时空间，例如基于磁盘的归并排序
- 为了从崩溃中恢复而进行的长时间计算的检查点
- 保存内容管理器容器提取的文件，同时Web服务器容器提供数据

#### hostPath

`hostPath` 卷将主机节点文件系统中的文件或目录挂载到 Pod 中。这不是大多数 Pod 都需要的，但对于某些应用程序来说，它提供了一个强大的逃生口。

`hostPath` 卷存在许多安全风险，因此在可能的情况下最好避免使用 HostPath。当必须使用 HostPath 卷时，应将其范围限定为仅所需的文件或目录，并以只读方式挂载。

如果通过 AdmissionPolicy 限制 HostPath 访问特定目录，则必须要求 volumeMounts 使用 readOnly 挂载，以使策略生效。

用途：

- 与 DaemonSet 一起运行，例如，EFK Fluentd 挂载本地主机的日志目录以收集主机日志信息。
- 通过使用 `hostPath` 卷在特定节点上运行，可以获得高性能的磁盘 I/O。
- 运行需要访问 Docker 内部的容器；使用 `/var/lib/docker` 的 hostPath。
- 在容器中运行 cAdvisor；使用 `/sys` 的 hostPath。
- 允许 Pod 指定给定的 hostPath 是否应该在 Pod 运行之前存在，是否应该创建它以及它应该存在的内容。

### Storage Class

StorageClass 部署和实现的步骤如下：

- 创建 Kubernetes 集群和后端存储。
- 确保 Kubernetes 中的 provisioner/plugin 可用。
- 创建一个 StorageClass 对象并将其链接到后端存储。StorageClass 将自动创建相关的 PV。
- 创建一个 PVC 对象并将其链接到我们创建的 StorageClass。
- 部署一个 Pod 并使用 PVC 卷。

### PV

PV回收策略：

- 保留 (Retain)
- 删除 (Delete)
- 回收 (Recycle)

PV in-tree类型：

- hostPath
- local
- NFS
- CSI

### Access Modes

Access Modes（访问模式）中，`spec.accessModes` 定义了 PV 的挂载选项：

- ReadWriteOnce(RWO)：一个 PV 只能被一个读写模式的 PVC 挂载，类似于块设备。
- ReadWriteMany(RWM)：一个 PV 可以被多个读写模式的 PVC 挂载，例如 NFS。
- ReadOnlyMany(ROM)：一个 PV 可以被多个只读模式的 PVC 挂载。
- ReadWriteOncePod(RWOP)：只支持 CSI 类型的 PV，只能被单个 Pod 挂载。

一个 PV 只能设置一种选项。Pod 挂载 PVC，而不是 PV。
