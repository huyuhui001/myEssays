# 我的Kubernetes学习心得

作为一种开源的容器编排系统，Kubernetes 为运行在容器中的应用程序提供了一种管理方式。Kubernetes 具有很多强大的功能，例如自动化部署、负载均衡、自动扩展、自动恢复等。
公司内部的云平台也在从Cloud Foundry环境开始向基于Kubernetes的Kyma环境延展，再加上各种媒体对Kubernetes的介绍，是我开始了解Kubernetes的外部动因。
内部动因，则是源于2022年春节期间参加了公司内部的一周Kubernetes基础培训，因为授课内容是英语，所以有很多细节在培训中是get不到的，主要还是语言能力不够强。当时的目标只是跟着完成讲师课堂演示。
从3月开始，我在网上了参考了别人的Kubernetes的学习心得和路线图，决定以CKA（certificate of Kubernetes administration）认证作为当前学习的目标，利用B站的视频，参考官方文档，开始从头开始学习Kubernetes的基础知识。

下面学习 Kubernetes 时的一些心得体会：

1. 学习 Kubernetes 前，需要了解容器技术和 Docker，或者说要有容器化的基本概念和思想方法，因为 Kubernetes 是基于容器技术构建的。
2. 学习 Kubernetes 时，需要掌握其核心概念，例如 Pod、ReplicaSet、Deployment、Service 等。这些概念是理解 Kubernetes 的基础，也是后续实际应用中的重要部分。
3. 学习 Kubernetes 时，需要掌握 yaml 文件的编写方法，特别是那些基本资源的yaml文件，要能做到不借助帮助文档就能写出框架，否则考试时做不完题目的。
4. 学习 Kubernetes 时，需要掌握 kubectl 命令的使用，常用资源相关的命令，也要做到不借助帮助文档就能写出，否则考试时也是做不完题目的。
5. 学习 Kubernetes 时，需要掌握其网络和存储配置，例如 Service、Ingress、PersistentVolume、PersistentVolumeClaim 等，这些都是考试重点内容，要熟悉yaml特性，能按不同的要求进行拓展和变更。
6. 部署和管理 Kubernetes 集群也是一个重点，我是在阿里云上买了3个ECS作为实验环境。

下面是我学习CKA的笔记。完成这里面的练习，再熟练使用yaml文件和kubectl命令，通过考试没什么困难。
原始的[英文笔记](https://huyuhui001.github.io/mySite/k8s/)是基于当时参加公司培训的知识结构做的，现在把自己的英语翻译成自己的中文，可能读起来会觉得有点怪，见笑了。
也附上中文笔记的[md文件](https://github.com/huyuhui001/myEssays/tree/hjmain/cka)给大家参考。

（还在持续翻译中........）

安装

- [CKA自学笔记1:单节点虚拟机安装Kubernetes](https://zhuanlan.zhihu.com/p/610095132)
- [CKA自学笔记2:多节点虚拟机安装Kubernetes](https://zhuanlan.zhihu.com/p/611350539)
- [CKA自学笔记3:阿里云ECS安装Kubernetes](https://zhuanlan.zhihu.com/p/612225376)

Docker

- [CKA自学笔记4:Docker基础](https://zhuanlan.zhihu.com/p/614388639)

基础知识

- [CKA自学笔记5:Kubernetes随笔](https://zhuanlan.zhihu.com/p/618940178)
- [CKA自学笔记6:Kubernetes集群概览](https://zhuanlan.zhihu.com/p/619623429)
- [CKA自学笔记7:kubectl基础](https://zhuanlan.zhihu.com/p/619683153)

核心概念

- [CKA自学笔记8:Pod](https://zhuanlan.zhihu.com/p/623856395)
- [CKA自学笔记9:Deployment](https://zhuanlan.zhihu.com/p/623856740)
- [CKA自学笔记10:Service](https://zhuanlan.zhihu.com/p/623856925)

应用体系

- [CKA自学笔记11:Namespace](https://zhuanlan.zhihu.com/p/623967582)
- [CKA自学笔记12:StatefulSet](https://zhuanlan.zhihu.com/p/623978015)
- [CKA自学笔记13:DaemonSet](https://zhuanlan.zhihu.com/p/623979116)
- [CKA自学笔记14:Job and Cronjob](https://zhuanlan.zhihu.com/p/623982580)
- [CKA自学笔记15:Configuration]
- [CKA自学笔记16:secrets]
- [CKA自学笔记17:Persistence](https://zhuanlan.zhihu.com/p/624313188)
- Role Based Access Control (RBAC)
- Ingress

进阶概念

- Scheduling
- Horizontal Pod Autoscaling
- Policy
- Network Policy
- Cluster Management

日常维护

- Troubleshooting
- Health Check
- Helming

案例

- Operations on Resources
- Health Check
- Calico Installation
