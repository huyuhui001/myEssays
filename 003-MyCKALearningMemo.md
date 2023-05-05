# 我的Kubernetes学习心得

作为一种开源的容器编排系统，Kubernetes 为运行在容器中的应用程序提供了一种管理方式。Kubernetes 具有很多强大的功能，例如自动化部署、负载均衡、自动扩展、自动恢复等。

公司内部的云平台也在从Cloud Foundry环境开始向基于Kubernetes的Kyma环境延展，再加上各种媒体对Kubernetes的介绍，是我开始了解Kubernetes的外部动因。

内部动因，则是源于2022年春节期间参加了公司内部的一周Kubernetes基础培训，因为授课内容是英语，所以有很多细节在培训中是get不到的，主要还是语言能力不够强。当时的目标只是跟着完成讲师课堂演示。

从3月开始，我在网上了参考了别人的Kubernetes的学习心得和路线图，决定以CKA（certificate of Kubernetes administration）认证作为当前学习的目标，利用B站的视频，参考官方文档，开始从头开始学习Kubernetes的基础知识。

下面准备CKA考试的一些心得体会：

* 学习 Kubernetes 前，需要了解容器技术和 Docker，或者说要有容器化的基本概念和思想方法，因为 Kubernetes 是基于容器技术构建的。
* 学习 Kubernetes 时，需要掌握其核心概念，例如 Pod、ReplicaSet、Deployment、Service 等。这些概念是理解 Kubernetes 的基础，也是后续实际应用中的重要部分。
* 学习 Kubernetes 时，需要掌握 yaml 文件的编写方法，特别是那些基本资源的yaml文件，要能做到不借助帮助文档就能写出框架，否则考试时做不完题目的。
* 学习 Kubernetes 时，需要掌握 kubectl 命令的使用，常用资源相关的命令，也要做到不借助帮助文档就能写出，否则考试时也是做不完题目的。
* 学习 Kubernetes 时，需要掌握其网络和存储配置，例如 Service、Ingress、PersistentVolume、PersistentVolumeClaim 等，这些都是考试重点内容，要熟悉yaml特性，能按不同的要求进行拓展和变更。
* 部署和管理 Kubernetes 集群也是一个重点，我是在阿里云上买了3个ECS作为实验环境。

我的CKA的笔记分中文和英文两种。英文笔记是基于第一次参加公司培训的知识结构做的，在备考过程中逐步完善的。中文笔记是在2023年4月份基于英文笔记自己翻译过来的，并发布在我的知乎专栏上，翻译过程还是比较难的，很多英语内容找不到合适的中文表达方式，不过对于计算机行业来讲，使用英语阅读专业资料应该是一个共识。

参考笔记，并完成笔记中的练习，再熟练使用yaml文件和kubectl命令，通过考试没什么困难。

**资料链接：**

* [知乎](https://zhuanlan.zhihu.com/p/624388855)
* [github](https://huyuhui001.github.io/mySite/k8s/)

**笔记目录：**

* CKA学习笔记:
  * 安装:
    * 单节点虚拟机安装Kubernetes
    * 多节点虚拟机安装Kubernetes
    * 阿里云ECS安装Kubernetes
  * Docker:
    * Docker基础
  * 基础知识:
    * Kubernetes随笔
    * Kubernetes集群概览
    * kubectl基础
  * 核心概念:
    * Pod
    * Deployment
    * Service
  * 应用体系:
    * Namespace
    * StatefulSet
    * DaemonSet
    * Job and Cronjob
    * Configuration
    * secrets
    * Persistence
    * RBAC鉴权
    * Ingress-nginx
  * 进阶概念:
    * Scheduling
    * Horizontal Pod Autoscaling (HPA)
    * Policy
    * Network Policy
    * Cluster Management
  * 日常维护:
    * Troubleshooting
    * 健康检查
    * Helm Chart
  * 主题讨论:
    * Kubernetes资源常见操作
    * 健康检查
    * 安装Calico
