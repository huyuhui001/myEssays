# Helm Chart

## Install Helm

Install Helm on `cka001`. 
```console
# https://github.com/helm/helm/releases
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
cp linux-amd64/helm /usr/bin/
rm -rf linux-amd64 helm-v3.8.2-linux-amd64.tar.gz
```

Or manually download the file via link `https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz`, and remote copy to `cka001`.
```console
scp -i cka-key-pair.pem ./Package/helm-v3.8.2-linux-amd64.tar.gz root@cka001:/root/
```
```console
ssh -i cka-key-pair.pem root@cka001
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
cp linux-amd64/helm /usr/bin/
rm -rf linux-amd64 helm-v3.8.2-linux-amd64.tar.gz
```



## Usage of Helm

Check `helm` version
```console
helm version
```
```
version.BuildInfo{Version:"v3.8.2", GitCommit:"6e3701edea09e5d55a8ca2aae03a68917630e91b", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

Get help of `helm`.
```console
helm help
```

Configure auto-completion for `helm`.
```console
echo "source <(helm completion bash)" >> ~/.bashrc
source <(helm completion bash)
```



## Install MySQL from Helm

Add bitnami Chartes Repository.
```console
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Get current Charts repositories.
```console
helm repo list
```
```
NAME    URL
bitnami https://charts.bitnami.com/bitnami
```

Sync up local Charts repositories.
```console
helm repo update
```

Search bitnami Charts in repositories.
```console
helm search repo bitnami
```

Search bitnami/mysql Charts in repositories.
```console
helm search repo bitnami/mysql
```

Install MySQL Chart on namespace `dev`：
```console
helm install mysql bitnami/mysql -n dev
```
Output
```
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

Check installed release：
```console
helm list
```
Result
```
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
mysql   dev             1               2022-07-24 19:37:20.710988009 +0800 CST deployed        mysql-9.2.1     8.0.29 
```

Check installed mysql release information.
```console
helm status mysql
```

Check mysql Pod status.
```console
kubectl get pod
```
Result
```
NAME                                      READY   STATUS    RESTARTS   AGE
mysql-0                                   1/1     Running   0          72s
```


## Develop a Chart

Below is a demo on how to develop a Chart.

Execute `helm create` to initiate a Chart：

```console
# Naming conventions of Chart: lowercase a~z and -(minus sign)
helm create cka-demo
```

A folder `cka-demo` was created. Check the folder structure.
```console
tree cka-demo/
```
Output
```
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

Delete or empty some files, which will be re-created later.
```console
cd cka-demo
rm -rf charts
rm -rf templates/tests 
rm -rf templates/*.yaml
echo "" > values.yaml
echo "" > templates/NOTES.txt
echo "" > templates/_helpers.tpl
cd ..
```

Now new structure looks like below.
```console
tree cka-demo/
```
Output
```
cka-demo/
├── Chart.yaml
├── templates
│   ├── _helpers.tpl
│   └── NOTES.txt
└── values.yaml
```


## NOTES.txt

NOTES.txt is used to provide summary information to Chart users. 
In the demo, we will use NOTES.txt to privide summary info about whether the user passed CKA certificate or not.
```console
cd cka-demo/
vi templates/NOTES.txt
```
Add below info.
```
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



## Deployment Template

Let's use Busybox service to generate information. 
We use `kubectl create deployment --dry-run=client -oyaml` to generate Deployment yaml file and write it the yaml file content into file `templates/deployment.yaml`.
```console
kubectl create deployment cka-demo-busybox --image=busybox:latest --dry-run=client -oyaml > templates/deployment.yaml
```

Check content of deployment yaml file `templates/deployment.yaml`.
```console
cat templates/deployment.yaml
```
```
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

Edit file `templates/deployment.yaml`.
```console
vi templates/deployment.yaml
```
Let's replace value of `.spec.replicas` from `1` to a variable `{{ .Values.replicaCount }}`, so we can dynamicly assign replicas number for other Deployment.
```
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

The `.spec.replicas` will be replaced by actula value of `.Values.replicaCount` during deployment. 

Let's create another file `values.yaml` and add a variable `replicaCount` with default value 1 into the file.
Strong recommended to add comments for each value we defined in file `values.yaml`.
```console
vi values.yaml
```
```
# Number of deployment replicas
replicaCount: 1
```

Let's add more variables into file `templates/deployment.yaml`.

* Replace Release name `.metadata.name` by `{{ .Release.Name }}` and filled with variable defined in file `values.yaml`.
* Replace label name `.metadata.labels` by `{{- include "cka-demo.labels" . | nindent 4 }}`, and filled with labels name `cka-demo.labels` defined in file `_helpers.tpl`.
* Replace `.spec.replicas` by `{{ .Values.replicaCount }}` and filled with variable defined in file `values.yaml`.
* Replace `.spec.selector.matchLabels` by `{{- include "cka-demo.selectorLabels" . | nindent 6 }}` and filled with `cka-demo.selectorLabels` defined in file `_helpers.tpl`.
* Replace `.spec.template.metadata.labels` by `{{- include "cka-demo.selectorLabels" . | nindent 8 }}` and filled with `cka-demo.selectorLabels` defined in file `_helpers.tpl`.
* Replace `.spec.template.spec.containers[0].image` by `{{ .Values.image.repository }}` and `{{ .Values.image.tag }}` and filled with variables defined in `values.yaml` for image name and image tag.
* Replace `.spec.template.spec.containers[0].command` and add `if-else` statement, if `.Values.passExam` is true, execute commands defined in `.Values.passCommand`, if false, execute commands defined in `.Values.lostCommand`.
* Use `key` from `ConfigMap` from `.spec.template.spec.containers[0].env` as prefix of ConfigMap name and filled with `{{ .Values.studentName }}` defined in file `values.yaml`.
* Replace `.spec.template.spec.containers[0].resources` by `{{ .Values.resources }}` and filled with variable defined in file `values.yaml`.

The `.Release.Name` is built-in object, no need to be specified in file `values.yaml`. It's generated by Release by `helm install`.


Remove unused lines and final one looks like below.
```
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



Update file `values.yaml` with variables default values.
Suggestions：add variables one and test one, don't add all at one time.
```console
vi values.yaml
```
```
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





## ConfigMap Template

ConfigMap is referred in the Deployment, hence we need define the ConfigMap template.
We will combine name of ConfigMap and `cka_score` as a variable, like `name-cka-score`.

```console
vi templates/configmap.yaml
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.studentName }}-cka-score
  labels:
    {{- include "cka-demo.labels" . | nindent 4 }}
data:
  cka_score: {{ .Values.ckaScore | quote }}
```

The `studentName` was already defined in file `values.yaml`, we just need add `ckaScore` with default value.
```console
vi values.yaml
```
```
# Student CKA Score
ckaScore: 100
```



## _helpers.tpl

Define a common template `_helpers.tpl` to add labels and labels of Selector for labels of Deployment and ConfigMap.
```console
vi templates/_helpers.tpl
```
```
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

We use CKA logo as the icon of Chart
```console
wget https://www.cncf.io/wp-content/uploads/2021/09/kubernetes-cka-color.svg
```

Edit Chart.yaml file.
```console
vi Chart.yaml
```
Append icon info in the file.
```
icon: file://./kubernetes-cka-color.svg
```

Add author info for the Chart
```console
vi Chart.yaml
```
```
maintainers:
  - name: James.H
```

Final `Chart.yaml` looks like below. Don't forget to update `appVersion: "v1.23"` to current Kubernetes API version.
```
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

Use `helm lint` to verify above change.
```console
helm lint
```
```
1 chart(s) linted, 0 chart(s) failed
```

`helm lint` only check format of Chart, won't check Manifest file.

We can use `helm install --debug --dry-run` or `helm template` to check Manifest output in order to verify all yaml files are correct or not.
```
helm template cka-demo ./
```

Use `helm install --debug --dry-run` to simulate the installation. We can get expected results from two different options (passed or failed the CKA certificate).
```console
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

Package Chart to .tgz file, and upload to repository, e.g., Chart Museum or OCI Repo.
```console
cd ../
helm package cka-demo
```
```
Successfully packaged chart and saved it to: /root/cka-demo-0.1.0.tgz
```

Till now, we have done our task to develop a Chart. Let's install the Chart.
```console
helm install cka-demo cka-demo-0.1.0.tgz --create-namespace \
  -n cka \
  --set studentName=kubernetes \
  --set ckaScore=0 \
  --set passExam=false
```
Result
```
NAME: cka-demo
LAST DEPLOYED: Sun Jul 24 19:58:36 2022
NAMESPACE: cka
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Come on! you can do it next time!
```

Check the deployment
```console
helm list --all-namespaces
```
Result
```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
cka-demo        cka             1               2022-07-24 19:58:36.272093383 +0800 CST deployed        cka-demo-0.1.0  v1.23      
mysql           dev             1               2022-07-24 19:37:20.710988009 +0800 CST deployed        mysql-9.2.1     8.0.29  
```

If any error, need to unstall `cka-demo` and reinstall it.
```console
helm uninstall cka-demo -n <your_namespace>
```

Check log of `cka-demo `.
```console
kubectl logs -n cka -l app=cka-demo
```
Result
```
Your CKA score is 0, Come on! you can do it next time!
```

Install `cka-demo` with different options.
```console
helm uninstall cka-demo -n cka

helm install cka-demo cka-demo-0.1.0.tgz --create-namespace \
  -n cka \
  --set studentName=kubernetes \
  --set ckaScore=100 \
  --set passExam=true
```
```
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

Check log of `cka-demo `.
```console
kubectl logs -n cka -l app=cka-demo
```
```
Your CKA score is 100 and your CKA certificate ID number is BQKoVYVhjzl3G
```




!!! Built-in Objects
    ```
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



!!! Reference
    * [Helm 官网](https://helm.sh/)
    * [Helm 版本支持策略](https://helm.sh/zh/docs/topics/version_skew/)
    * [Helm Chart 资源对象安装顺序](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go)



