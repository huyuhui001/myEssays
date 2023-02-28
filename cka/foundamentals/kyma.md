# Kyma

Deploy Kyma on control plane node.

## Install Kyma CLI

Install Kyma CLI on Linux, run:
```
curl -Lo kyma.tar.gz "https://github.com/kyma-project/cli/releases/download/$(curl -s https://api.github.com/repos/kyma-project/cli/releases/latest | grep tag_name | cut -d '"' -f 4)/kyma_Linux_x86_64.tar.gz"
mkdir /opt/kyma-release
tar -C /opt/kyma-release -zxvf kyma.tar.gz
chmod +x /opt/kyma-release/kyma
sudo cp /opt/kyma-release/kyma /usr/local/bin
rm -rf kyma-release kyma.tar.gz
```

!!! Reference
    [Install Kyma CLI](https://kyma-project.io/docs/kyma/latest/04-operation-guides/operations/01-install-kyma-CLI/)


## Install Kyma

Use the deploy command to install Kyma.
```
kyma deploy
```

Get file [components.yaml](https://github.com/kyma-project/kyma/blob/main/installation/resources/components.yaml) to manually install failed components. If no Namespace provided, the default Namespace called `kyma-system` is used. 
For example:
```
kyma deploy --component serverless@kyma-integration
kyma deploy --component monitoring@kyma-integration
kyma deploy --component kiali@kyma-integration
```

File `components.yaml` looks like below.
```
---
defaultNamespace: kyma-system
prerequisites:
  - name: "cluster-essentials"
  - name: "istio"
    namespace: "istio-system"
  - name: "certificates"
    namespace: "istio-system"
components:
  - name: "istio-resources"
  - name: "logging"
  - name: "telemetry"
  - name: "tracing"
  - name: "kiali"
  - name: "monitoring"
  - name: "eventing"
  - name: "ory"
  - name: "api-gateway"
  - name: "cluster-users"
  - name: "serverless"
  - name: "application-connector"
    namespace: "kyma-integration"
```

!!! Reference    
    By default, Kyma is installed with the default chart values defined in the values.yaml files. You can also control the allocation of resources, such as memory and CPU, that the components consume by installing Kyma with the following pre-defined profiles:
    
    * Evaluation needs limited resources and is suited for trial purposes.
    * Production is configured for high availability and scalability. It requires more resources than the evaluation profile but is a better choice for production workload.

    [Install Kyma](https://kyma-project.io/docs/kyma/latest/04-operation-guides/operations/02-install-kyma/)

    To see a complete list of all Kyma components go to the [components.yaml](https://github.com/kyma-project/kyma/blob/main/installation/resources/components.yaml) file. Install specific components `kyma deploy --components-file {COMPONENTS_FILE_PATH}`




















