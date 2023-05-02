# CKA自学笔记27:Helm Chart

## 安装Helm

在节点`cka001`上安装Helm。

```bash
# https://github.com/helm/helm/releases
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
cp linux-amd64/helm /usr/bin/
rm -rf linux-amd64 helm-v3.8.2-linux-amd64.tar.gz
```

或者从链接 `https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz` 手工下载安装包，并拷贝到节点 `cka001`上。

```bash
scp -i cka-key-pair.pem ./Package/helm-v3.8.2-linux-amd64.tar.gz root@cka001:/root/
```

```bash
ssh -i cka-key-pair.pem root@cka001
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
cp linux-amd64/helm /usr/bin/
rm -rf linux-amd64 helm-v3.8.2-linux-amd64.tar.gz
```

## Helm用法

检查 `helm` 的版本。

```bash
helm version
```

运行结果：

```console
version.BuildInfo{Version:"v3.8.2", GitCommit:"6e3701edea09e5d55a8ca2aae03a68917630e91b", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

获取 `helm` 的帮助信息。

```bash
helm help
```

配置 `helm` 的命令自动补全功能。

```bash
echo "source <(helm completion bash)" >> ~/.bashrc
source <(helm completion bash)
```

## 通过Helm安装MySQL

添加Bitnami Chartes仓库。

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

查询当前可用的Chartes仓库。

```bash
helm repo list
```

运行结果：

```console
NAME    URL
bitnami https://charts.bitnami.com/bitnami
```

同步本地Charts仓库。

```bash
helm repo update
```

在Charts仓库中查找bitnami Charts仓库。

```bash
helm search repo bitnami
```

在仓库中搜索bitnami/mysql Charts。

```bash
helm search repo bitnami/mysql
```

在namespace `dev`上安装MySQL Chart。

```bash
helm install mysql bitnami/mysql -n dev
```

运行结果：

```console
NAME: mysql
LAST DEPLOYED: Sun Jul 24 19:37:20 2022
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.2.1
APP VERSION: 8.0.29

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace dev

Services:

  echo Primary: mysql.dev.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace dev mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.29-debian-11-r9 --namespace dev --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql.dev.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

查看当前安装包的信息。

```bash
helm list
```

运行结果：

```console
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
mysql   dev             1               2022-07-24 19:37:20.710988009 +0800 CST deployed        mysql-9.2.1     8.0.29 
```

检查当前安装的mysql版本信息。

```bash
helm status mysql
```

检查pod mysql 的状态。

```bash
kubectl get pod
```

运行结果：

```console
NAME                                      READY   STATUS    RESTARTS   AGE
mysql-0                                   1/1     Running   0          72s
```

## 部署一个Chart

下面演示了如何部署一个Chart。

执行命令 `helm create` 来初始化一个Chart。

```bash
# Naming conventions of Chart: lowercase a~z and -(minus sign)
helm create cka-demo
```

目录 `cka-demo` 会被创建，查看这个目录的结构。

```bash
tree cka-demo/
```

运行结果：

```console
cka-demo/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

删除或清空某些文件，我们会在后面重新创建这些文件。

```bash
cd cka-demo
rm -rf charts
rm -rf templates/tests 
rm -rf templates/*.yaml
echo "" > values.yaml
echo "" > templates/NOTES.txt
echo "" > templates/_helpers.tpl
cd ..
```

目录 `cka-demo` 的架构现在应该看起来类似下面的结果。

```bash
tree cka-demo/
```

运行结果：

```console
cka-demo/
├── Chart.yaml
├── templates
│   ├── _helpers.tpl
│   └── NOTES.txt
└── values.yaml
```

## NOTES.txt

NOTES.txt 用于向 Chart 用户提供概要信息。在演示中，我们将使用 NOTES.txt 提供关于用户是否通过 CKA 认证的概要信息。

```bash
cd cka-demo/
vi templates/NOTES.txt
```

添加下面的内容。

```console
{{- if .Values.passExam }}
Congratulations!

You have successfully completed Certified Kubernetes Administrator China Exam (CKA-CN). 

Your CKA score is: {{ .Values.ckaScore }}

Click the link below to view and download your certificate.

https://trainingportal.linuxfoundation.org/learn/dashboard
{{- else }}
Come on! you can do it next time!
{{- end }}
```

## 部署模版

下面会使用 Busybox 服务来生成信息。
通过命令 `kubectl create deployment --dry-run=client -oyaml` 生成 Deployment 的 YAML 文件，并将其内容写入文件 `templates/deployment.yaml`。

```bash
kubectl create deployment cka-demo-busybox --image=busybox:latest --dry-run=client -oyaml > templates/deployment.yaml
```

检查deployment的yaml文件`templates/deployment.yaml`的内容。

```bash
cat templates/deployment.yaml
```

运行结果：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: cka-demo-busybox
  name: cka-demo-busybox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cka-demo-busybox
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: cka-demo-busybox
    spec:
      containers:
      - image: busybox:latest
        name: busybox
        resources: {}
status: {}
```

编辑修改文件 `templates/deployment.yaml`。

```bash
vi templates/deployment.yaml
```

让我们将 `.spec.replicas` 的值从 `1` 替换为变量 `{{ .Values.replicaCount }}`，这样我们可以为其他 Deployment 动态分配副本数。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: cka-demo-busybox
  name: cka-demo-busybox
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: cka-demo-busybox
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: cka-demo-busybox
    spec:
      containers:
      - image: busybox:latest
        name: busybox
        resources: {}
status: {}
```

`.spec.replicas` 将在部署期间被实际的 `.Values.replicaCount` 值替换。

现在创建另一个文件 `values.yaml` 并在文件中添加一个变量 `replicaCount`，默认值为1。

强烈建议在文件 `values.yaml` 中定义的每个值添加注释。

```bash
vi values.yaml
```

输出结果：

```console
# Number of deployment replicas
replicaCount: 1
```

下面对文件 `templates/deployment.yaml` 添加更多的变量。

* 将 `.metadata.name` 的 Release 名称替换为 `{{ .Release.Name }}`，并用在 `values.yaml` 文件中定义的变量填充。
* 将标签名称 `.metadata.labels` 替换为 `{{- include "cka-demo.labels" . | nindent 4 }}`，并用在 `_helpers.tpl` 文件中定义的标签名称 `cka-demo.labels` 填充。
* 将 `.spec.replicas` 替换为 `{{ .Values.replicaCount }}`，并用在 `values.yaml` 文件中定义的变量填充。
* 将 `.spec.selector.matchLabels` 替换为 `{{- include "cka-demo.selectorLabels" . | nindent 6 }}` 并使用在 `_helpers.tpl` 文件中定义的 `cka-demo.selectorLabels` 进行填充。
* 将 `.spec.template.metadata.labels` 替换为 `{{- include "cka-demo.selectorLabels" . | nindent 8 }}` 并使用在 `_helpers.tpl` 文件中定义的 `cka-demo.selectorLabels` 进行填充。
* 将 `.spec.template.spec.containers[0].image` 替换为 `{{ .Values.image.repository }}` 和 `{{ .Values.image.tag }}` 并使用在 `values.yaml` 文件中定义的变量填充镜像名称和镜像标签。
* 将 `.spec.template.spec.containers[0].command` 替换为一个 `if-else` 语句，如果 `.Values.passExam` 为真，则执行在 `.Values.passCommand` 中定义的命令，否则执行在 `.Values.lostCommand` 中定义的命令。
* 使用 `.spec.template.spec.containers[0].env` 中的 `key` 作为 ConfigMap 名称的前缀，并使用在 `values.yaml` 文件中定义的 `{{ .Values.studentName }}` 进行填充。
* 将 `.spec.template.spec.containers[0].resources` 替换为 `{{ .Values.resources }}` 并使用在 `values.yaml` 文件中定义的变量进行填充。

`.Release.Name`是内置对象，在文件`values.yaml`中不需要指定。它是由`helm install`生成的Release。

移除不必要的行，最终文件看起来类似下面的结果：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "cka-demo.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "cka-demo.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "cka-demo.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: id-generator
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        {{- if .Values.passExam }}
        {{- with .Values.passCommand }}
        command: {{ range . }}
          - {{ . | quote }}
          {{- end }}
          {{- end }}
        {{- else }}
        {{- with .Values.lostCommand }}
        command: {{ range . }}
          - {{ . | quote }}
          {{- end }}
          {{- end }}
        {{- end }}
        env:
        - name: CKA_SCORE
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.studentName }}-cka-score
              key: cka_score
        {{- with .Values.resources }}
        resources:
            {{- toYaml . | nindent 12 }}
          {{- end}}
      restartPolicy: Always
```

更新文件 `values.yaml` 中变量的默认值。建议逐个添加变量并测试，不要一次添加所有变量。

```bash
vi values.yaml
```

运行结果：

```console
# Number of deployment replicas 
replicaCount: 1

# Image repository and tag
image:
  repository: busybox
  tag: latest

# Container start command
passCommand:
  - '/bin/sh'
  - '-c'
  - "echo Your CKA score is $(CKA_SCORE) and your CKA certificate ID number is $(tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 13; echo) ; sleep 86400"
lostCommand:
  - '/bin/sh'
  - '-c'
  - "echo Your CKA score is $(CKA_SCORE), Come on! you can do it next time! ; sleep 86400"

# Container resources
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
    
# Student Name
studentName: whoareyou

# Student pass CKA exam or not
passExam: true
```

## ConfigMap模版

ConfigMap被部署中的Deployment所引用，因此我们需要定义ConfigMap的模板。
我们将把ConfigMap的名称和`cka_score`组合成一个变量，例如`name-cka-score`。

```bash
vi templates/configmap.yaml
```

运行结果：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.studentName }}-cka-score
  labels:
    {{- include "cka-demo.labels" . | nindent 4 }}
data:
  cka_score: {{ .Values.ckaScore | quote }}
```

`studentName`已经在`values.yaml`文件中定义过了，我们只需要添加一个名为`ckaScore`的变量并给它一个默认值即可。

```bash
vi values.yaml
```

运行结果

```console
# Student CKA Score
ckaScore: 100
```

## _helpers.tpl

定义一个通用的模板`_helpers.tpl`，为Deployment和ConfigMap的标签和选择器标签添加标签。

```bash
vi templates/_helpers.tpl
```

运行结果：

```console
{{/*
Common labels
*/}}
{{- define "cka-demo.labels" -}}
{{ include "cka-demo.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}


{{/*
Selector labels
*/}}
{{- define "cka-demo.selectorLabels" -}}
app: {{ .Chart.Name }}
release: {{ .Release.Name }}
{{- end -}}
```

## Chart.yaml

这里我们使用CKA的logo来作为Chart的图标。

```bash
wget https://www.cncf.io/wp-content/uploads/2021/09/kubernetes-cka-color.svg
```

编辑修改 Chart.yaml 文件。

```bash
vi Chart.yaml
```

把图标信息添加到Chart.yaml文件末尾。

```console
icon: file://./kubernetes-cka-color.svg
```

把作者信息添加到Chart.yaml文件末尾。

```bash
vi Chart.yaml
```

运行结果：

```console
maintainers:
  - name: James.H
```

最终的 `Chart.yaml` 类似如下内容。别忘记更新 `appVersion: "v1.23"` 为当前Kubernetes的版本。

```console
apiVersion: v2
name: cka-demo
description: A Helm chart for CKA demo.
type: application
version: 0.1.0
appVersion: "v1.23"
maintainers:
  - name: James.H
icon: file://./kubernetes-cka-color.svg
```

## Chart Debug

使用 `helm lint` 来验证上述变更。

```bash
helm lint
```

运行结果：

```console
1 chart(s) linted, 0 chart(s) failed
```

`helm lint` 只检查Chart的格式，不检查Manifest文件。

我们可以使用 `helm install --debug --dry-run` 或 `helm template` 命令来检查生成的 Manifest 是否正确。

```bash
helm template cka-demo ./
```

通过命令 `helm install --debug --dry-run`来模拟安装。我们可以从两个不同的选项（通过或未通过CKA认证）中获得预期的结果。

```bash
helm install --debug --dry-run cka-demo ./ --create-namespace \
  -n cka \
  --set studentName=kubernetes \
  --set ckaScore=99 \
  --set passExam=true
  
helm install --debug --dry-run cka-demo ./ --create-namespace \
  -n cka \
  --set studentName=kubernetes \
  --set ckaScore=0 \
  --set passExam=false
```

把 Chart 打包成 `.tgz` 文件，并上传到仓库，例如 Chart Museum 或者 OCI Repo。

```bash
cd ../
helm package cka-demo
```

运行结果：

```console
Successfully packaged chart and saved it to: /root/cka-demo-0.1.0.tgz
```

至此，我们已经完成了配置一个新的Chart，现在开始安装这个Chart。

```bash
helm install cka-demo cka-demo-0.1.0.tgz --create-namespace \
  -n cka \
  --set studentName=kubernetes \
  --set ckaScore=0 \
  --set passExam=false
```

运行结果：

```console
NAME: cka-demo
LAST DEPLOYED: Sun Jul 24 19:58:36 2022
NAMESPACE: cka
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Come on! you can do it next time!
```

检查部署情况：

```bash
helm list --all-namespaces
```

运行结果：

```console
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
cka-demo        cka             1               2022-07-24 19:58:36.272093383 +0800 CST deployed        cka-demo-0.1.0  v1.23      
mysql           dev             1               2022-07-24 19:37:20.710988009 +0800 CST deployed        mysql-9.2.1     8.0.29  
```

如果遇到错误，则需要卸载`cka-demo`并重新安装它。

```bash
helm uninstall cka-demo -n <your_namespace>
```

检查`cka-demo`的日志。

```bash
kubectl logs -n cka -l app=cka-demo
```

运行结果

```console
Your CKA score is 0, Come on! you can do it next time!
```

通过其他选项安装 `cka-demo`。

```bash
helm uninstall cka-demo -n cka

helm install cka-demo cka-demo-0.1.0.tgz --create-namespace \
  -n cka \
  --set studentName=kubernetes \
  --set ckaScore=100 \
  --set passExam=true
```

运行结果：

```console
NAME: cka-demo
LAST DEPLOYED: Sun Jul 24 20:01:34 2022
NAMESPACE: cka
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Congratulations!

You have successfully completed Certified Kubernetes Administrator China Exam (CKA-CN).

Your CKA score is: 100

Click the link below to view and download your certificate.

https://trainingportal.linuxfoundation.org/learn/dashboard
```

检查 `cka-demo` 的日志。

```bash
kubectl logs -n cka -l app=cka-demo
```

运行结果：

```console
Your CKA score is 100 and your CKA certificate ID number is BQKoVYVhjzl3G
```

Built-in Objects列表

```console
Release.Name                              # 发布名称
Release.Namespace                         # 发布Namespace
Release.Service                           # 渲染模板的服务，在Helm中默认值为"Helm"
Release.IsUpgrade                         # 如果当前是升级或回滚，设置为true
Release.IsInstall                         # 如果当前是安装，设置为true
Release.Revision                          # 发布版本号
Values                                    # 从values.yaml和--set传入，默认为空
Chart                                     # 所有Chart.yaml中的内容
Chart.Version                             #
Chart.Maintainers                         #
Files                                     # 在chart中访问非特殊文件
Capabilities                              # 提供关于支持能力的信息（K8s API版本、K8s版本、Helm版本）
Capabilities.KubeVersion                  # Kubernetes的版本号
Capabilities.APIVersions.Has "batch/v1"   # K8s API版本包含"batch/v1"
Template                                  # 当前模板信息
Template.Name                             # 当前模板文件路径
Template.BasePath                         # 当前模板目录路径
```

参考：

* [Helm 官网](https://helm.sh/)
* [Helm 版本支持策略](https://helm.sh/zh/docs/topics/version_skew/)
* [Helm Chart 资源对象安装顺序](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go)
