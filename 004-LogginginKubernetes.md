# Kubernetes的日志记录(logging)

## 目的

了解不同类型的Kubernetes日志和日志管道。

## 1.简介

Kubernetes是一个开源的容器编排器，旨在管理和扩展应用程序。在使用Kubernetes时，开发人员和DevOps工程师需要知道如何通过不同类型的日志来调试集群并查找问题。由于Kubernetes的动态特性，对于监控来说，日志的集中化是一个具有挑战性的任务。

我们都知道，日志是列出已发生事件的文件。它涉及与计算机操作系统或运行在该系统上的软件相关的过去操作的详细信息。下面的内容会讨论Kubernetes中的日志记录解决方案。

下面会简要介绍了什么是Kubernetes日志记录？它是如何工作的？为什么需要它？Kubernetes中的不同类型的日志和广泛使用的日志记录工具，等。

## 2.日志记录是什么？

在一般计算中，日志记录意味着在计算机系统中对事件或操作进行的书面记录。例如，在应用程序级别，当发生或影响应用程序的事件时，它会生成日志。

## 3. Kubernetes中日志记录的重要性

日志记录对于收集关键信息进行调试和许多其他方面非常重要，例如：

- 快速识别问题和提前发现
- 监控应用程序，节点和集群的性能
- 提供安全性，例如保护免受未经授权访问
- 知道发生在Kubernetes集群中的一系列操作
- 提高集群操作的效率

## 4. Kubernetes中的日志记录架构

为了从Kubernetes集群收集日志，我们需要从以下地方收集日志：

- 所有部署在Pod上的应用程序的日志
- 所有节点的日志
- Kubernetes控制平面组件和工作节点的所有日志

目前没有原生的日志记录解决方案可以涵盖我们上面提到的所有内容，但是我们可以通过以下方式实现。

- 日志记录代理：通常在集群中以DaemonSet的形式部署，用于收集节点和应用程序级别的日志。然后将聚合日志发送到集中的位置，例如FluentD，Logstash等。
- 集中式后端：它存储、索引、搜索和分析日志数据，例如Elasticsearch，Grafana Loki等。
- 可视化：用于在仪表板上可视化日志数据，例如Grafana，Kibana等。

### 4.1.应用程序级/ Pod级别的日志

在Kubernetes集群内部运行的应用程序的日志可以使用kubectl查看。下面举一个例子。

- 创建一个nginx web服务器Pod并观察其日志。

```bash
kubectl run webapp --image=nginx
kubectl logs webapp
```

如果Pod中运行多个容器，使用`-c`标志指定容器的名称。

我们可以检查被发送到`stdout`或`stderr`的Pod的日志及其容器，它们分别存储在节点的`/var/log/pods`和`/var/log/containers`目录中。

如果容器重新启动，`kubelet`负责在节点级别跟踪日志。而且，当Pod从节点上驱逐时，其容器和相应的日志也会被驱逐。

- 列出节点上每个Pod的日志文件。

```bash
ls -l /var/log/pods
```

- 检查任何Pod及其相应容器的日志。

```bash
cat <namespace_pod_name>/<container_name>/<file-name>.log
```

`/var/log/containers`目录中的文件是指向它们相应的`/var/log/pods/<namespace_podname>`文件夹的符号链接。

- 列出节点上每个运行中容器的日志文件。

```bash
ls -l /var/log/containers
```

### 4.2.节点级别的日志

节点级别的日志与应用程序级别的日志不同，因为它还包含所有与节点基础设施相关的信息。它将包含以下每个节点的信息：

- 内核日志
- Systemd日志
- 其他操作系统日志

要使用`journald`查看节点级别的内核和Systemd日志，可以运行以下命令。

```bash
journalctl
```

要查看特定服务的日志，例如`kubelet`（它也是Kubernetes每个节点组件的一部分）的日志，可以运行以下命令。

```bash
journalctl -fu kubelet
```

要获取其他无法从上述两个位置检查的系统级别日志，可以在`/var/log/location`中找到。例如，列出nginx web服务器的访问和错误日志。

```bash
ls /var/log/nginx
```

获取`access.log`文件的内容。

```bash
cat /var/log/nginx/access.log
```

获取`error.log`文件的内容。

```bash
cat /var/log/nginx/error.log
```

我们可以使用日志代理，如FluentD或Logstash来收集节点级别和应用程序级别的日志，这些代理可以访问所有日志文件和目录。我们可以根据用例在特定节点上运行这些代理服务，或者作为DaemonSet在集群的所有节点上运行。

### 4.3.集群级别的日志

集群级别的日志是来自以下内容的日志：

- 运行在Pods内的应用程序
- 集群的节点
- 控制平面组件

前两者我们在上面已经讨论过了。由于Kubernetes集群组件作为Pods运行，日志收集器代理将收集API Server和其他控制平面组件的日志；而`kubelet`日志则通过`systemd`来收集。

除了上述内容，我们可以使用以下内容获取更多详细信息：

- Kubernetes事件
- 审计日志

#### 4.3.1. Kubernetes事件

Kubernetes事件存储关于对象状态更改和错误的信息。事件存储在控制平面节点上的API服务器中。

获取事件日志

```bash
kubectl get events -n <namespace>
```

我们还可以从特定的Pod获取事件。

描述一个Pod

```bash
kubectl describe pod <pod-name>
```

在输出的末尾检查事件。

为了避免在控制平面节点上使用所有磁盘空间，Kubernetes在一个小时后删除事件。

#### 4.3.2. Kubernetes审计日志

审计日志是集群中发生的所有操作的详细信息。它以序列的方式跟踪所有活动。它用于安全目的，以了解谁在何时以及如何进行了哪些操作。审核策略必须在仅主控节点上启用。

## 5. 日志记录管道

日志记录管道由收集来自不同Pod应用程序的日志，将其聚合并发送到后端，其中存储日志的组件组成。然后用于进一步分析和可视化。日志记录管道进一步划分为：

- 日志记录库
- 日志记录代理和转发器
- 日志记录平台

### 5.1.日志记录库

日志记录库用于以预定义的格式获取日志，如文本、JSON等，通常是基于编程语言、内存利用偏好等选择的。一些日志记录库是Log4j、Zap等。

### 5.2.日志记录代理和转发器

日志记录代理和转发器帮助收集应用程序的日志并将其进一步转发到集中的位置。两者有一点差异，但可以互换使用。日志记录代理的例子包括FluentD、Logstash等，而转发器的例子包括FluentBit、Filebeat等。

### 5.3.日志记录平台

日志记录平台接收和获取来自日志记录代理/转发器的日志。它以可靠且安全的方式存储日志，并提供界面以进行分析。

不同的日志记录平台如下：

- Grafana Loki
- Elasticsearch
- Parseable

#### 5.3.1. Grafana Loki

Grafana Loki是一个开源的日志聚合系统，用于存储和查询来自应用程序和基础设施的日志。它高度可伸缩，可以处理大量的日志数据。它采用最小化索引的方法，只对日志的元数据进行索引。它将数据存储在高度压缩的块中，从而降低了成本。

特点：

- 多租户
- 使用LogQL，Loki自己的查询语言
- 可扩展性和灵活性
- 与Grafana集成进行可视化

#### 5.3.2. Elasticsearch

Elasticsearch是一个开源的、分布式的搜索和分析引擎，使用Java开发，构建在Apache Lucene之上。它是ELK堆栈（Elasticsearch、Logstash、Kibana）的重要组件之一。它是一个NoSQL数据库。

特点：

- 快速查询和分析
- 可扩展性
- 韧性
- 灵活性

#### 5.3.3. Parseable

Parseable是一个开源的、轻量级的、云原生的日志可观测引擎，使用Rust编写。它使用Apache Arrow和Parquet轻松访问数据。它使用无索引机制对数据进行组织和查询，延迟低。它可以在任何地方部署。它被设计成无状态的，以使其与云原生系统兼容。

特点：

- 使用本地卷或S3兼容对象存储
- 无状态和无索引
- 基于日志量自动缩放
- Parquet格式
- 无模式设计
- 简单数据访问（基于PostgreSQL的日志数据查询）
- 用于日志查看和可观察性的内置GUI

## 6.可视化

用户交互式数据可视化平台如Kibana、Grafana是强大的工具，可帮助观察系统的状态。通过日志可视化，用户可以轻松地识别和分析问题。用户可以通过不同类型的图表（如时间序列）在日志系统中检查特定的日志条目。这些可视化可以根据时间范围、用户权限等进行过滤。

## 7. Kubernetes日志记录的最佳实践

在使用Kubernetes日志记录工具时，确保以最小的资源利用率有效使用它。因此，以下是在Kubernetes中使用日志记录的一些建议：

- 设置日志记录系统
- 将日志写入`stdout`和`stderr`
- 使用标准日志库
- 使用日志轮转策略限制日志量
- 在开发和生产中使用独立的集群
