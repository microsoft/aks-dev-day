# Helm Lab

## Overview

In this Lab, you will use Helm to package and deploy an application on the AKS cluster.
[Helm](https://helm.sh/) is the package manager for Kubernetes.
Helm Charts help you define, install, and upgrade even the most complex Kubernetes application.
Charts are easy to create, version, share, and publish.

## Prerequisites

To perform this Lab, you'll need:

- An AKS cluster (created in previous labs)
- Azure CLI installed (Check with `az version`)
- [Helm v3 installed](https://helm.sh/docs/intro/install/) (Check with `helm version`)
- git client (Check with `git version`)

## Steps

### Create an Azure Container Registry

For this lab, you will store the application image in an Azure Container Registry, then pull this image from the AKS cluster when deploying with Helm.

The registry name must be globally unique, 5-50 alphanumeric characters.

```cmd
az acr create --resource-group rg-use-azint-aks-devdays --name acruseaksdevdays --sku Basic
```

In the output, take note of the login server. We will need it later.
From above, the login server is: `"loginServer": "acruseaksdevdays.azurecr.io",`

To ensure that the AKS cluster will be able to use this ACR, especially if Azure AD RBAC is used, run this command:

```cmd
az aks update -n aks-use-azint-aks-devdays -g rg-use-azint-aks-devdays --attach-acr acruseaksdevdays
```

### Prepare the application

We will clone the application locally and use the ACR to create and store its image:

```cmd
cd <any folder that works for you>
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
cd azure-voting-app-redis/azure-vote/
dir
```

Verify you see a file named `Dockerfile`.

We use the file to build the application image:

```cmd
az acr build --image azure-vote-front:v1 --registry acruseaksdevdays --file Dockerfile .
```

That shows you the ACR can be used directly to build images. It's a simpler process than building them locally, tagging them, and pushing them to an ACR.
Additionally, an ACR can import and manage Helm charts. For more information, see [Push and pull Helm charts to an Azure container registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-helm-repos).

### Create the application Helm chart

Generate the Helm chart for the app with:

```cmd
helm create azure-vote-front
```

This command creates a `azure-vote-front` folder with the Helm chart in it.
Go in it and look at the content structure ([The Chart File Structure](https://helm.sh/docs/topics/charts/#the-chart-file-structure)), as a chart is organized as a collection of files inside of a directory:

File | Comment | Reference
---------|----------|----------
 `Chart.yaml` | This file is the application manifest. It sets the version, dependencies, information on the app, etc. | [The Chart.yaml File](https://helm.sh/docs/topics/charts/#the-chartyaml-file)
 `.helmignore` | The .helmignore file is used to specify files you don't want to include in your helm chart. | [The .helmignore file](https://helm.sh/docs/chart_template_guide/helm_ignore_file/#helm)
 `values.yaml` | This file sets the default values of the Helm chart | [Values](https://helm.sh/docs/chart_best_practices/values/#helm)
 `charts/` | This folder contains any charts upon which this chart depends. It allows manual management of dependencies. The other way is to declare them in the `Chart.yaml` | [Chart dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies)
 `templates/` | A directory of templates - mainly `YAML` files - that, when combined with values, generate valid Kubernetes manifest files. | [Templates](https://helm.sh/docs/chart_best_practices/templates/#helm)

To understand, at a high-level, how Helm charts work, open the `templates/service.yaml` file.
You will see the mix of Kubernetes YAML declarations, and sections surrounded by `{{ ... }}`. These sections are anchors for Helm to generate the content based on values and operations.

### Adapt the Helm chart

- Update the `Chart.yaml` to add a dependency on the `redis` chart, by adding this declaration:

```yaml
dependencies:
  - name: redis
    version: 17.3.14
    repository: https://charts.bitnami.com/bitnami
```

Save the file and run:

`helm dependency update azure-vote-front`

Success looks like:

```cmd
Getting updates for unmanaged Helm repositories...
...Successfully got an update from the "https://charts.bitnami.com/bitnami" chart repository
Saving 1 charts
Downloading redis from repo https://charts.bitnami.com/bitnami
```

Additionally, you will see a `redis-17.3.14.tgz` file added to the `charts/` folder.

- Update the `values.yaml` to set the defaults for the chart:

  - Replace the default section:

    FROM:

    ```yaml
    replicaCount: 1

    image:
      repository: nginx
      pullPolicy: IfNotPresent
      # Overrides the image tag whose default is the chart appVersion.
      tag: ""
    ```

    TO:

    ```yaml
    replicaCount: 1
    backendName: azure-vote-backend-master
    redis:
      image:
        registry: mcr.microsoft.com
        repository: oss/bitnami/redis
        tag: 6.0.8
      fullnameOverride: azure-vote-backend
      auth:
        enabled: false

    image:
      repository: acruseaksdevdays.azurecr.io/azure-vote-front
      pullPolicy: IfNotPresent
      tag: "v1"
    ```

    **Important Note**: see that the repository for the image (`image.repository`) uses our ACR login server value noted earlier.

  - Change `service.type` to LoadBalancer:

      FROM:

      ```yaml
      service:
        type: ClusterIP
        port: 80
      ```

      TO:

      ```yaml
      service:
        type: LoadBalancer
        port: 80
      ```

- Update the `templates/deployment.yaml` to set the redis environment variable value:

    FROM:

    ```yaml
    ...
    imagePullPolicy: {{ .Values.image.pullPolicy }}
    ports:
    ...
    ```

    TO:

    ```yaml
    ...
    imagePullPolicy: {{ .Values.image.pullPolicy }}
    env:
    - name: REDIS
      value: {{ .Values.backendName }}
    ports:
    ...
    ```

### Test the Helm chart

To test all the changes did not break the Chart, run:

`helm lint azure-vote-front`

If any errors (and with YAML it's easy to have some with indentation), they will show.

Success looks like:
`1 chart(s) linted, 0 chart(s) failed`

### Install the Application in AKS

To deploy to an AKS cluster, Helm will leverage the credentials used by `kubectl` (since v3+).

Get/update the AKS cluster credentials:

- if `kubectl.exe` is not installed:
  `az aks install-cli`

- get credentials for the AKS cluster:
  `az aks get-credentials -n aks-use-azint-aks-devdays -g rg-use-azint-aks-devdays`

Install the Helm chart release:

`helm install azure-vote-front azure-vote-front/`

Success looks like:

```cmd
NAME: azure-vote-front
LAST DEPLOYED: Thu Dec  8 13:20:14 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

See the Release in the AKS Cluster:
`helm list`

See the deployment of the Application in AKS:
`kubectl get all`

Check the application is operational:

- run `kubectl get svc` to obtain the `EXTERNAL-IP` of the service `azure-vote-front`
- Browse to the IP with `http://<EXTERNAL-IP>`, until the `"Azure Voting App"` appears
- if `Cats` and `Dogs` add, then the redis cache is used and the application successfully deployed with all its components.

### Uninstall the Application in AKS

Uninstall the Helm chart release:

`helm uninstall azure-vote-front`

See all the deployed resources are cleared with:
`kubectl get all`
