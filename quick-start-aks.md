# Quick start

In this guide we will see how to start the [Initium Platform](https://github.com/nearform/initium-platform) on a AKS cluster and deploy an application to it from a GitHub action using the [Initium CLI](https://github.com/nearform/initium-cli).

## Install Azure CLI Locally
Ignore this step if you already have Azure CLI setup.

Install `azure-cli` on your machine by following the [Azure official guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli). 

## Login to Azure

Run the command below. Note that there are different ways one can log into Azure using the Azure CLI. Refer to the [az login docs](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli) if you have any questions.

For this guide, we'll use the option `Sign in interactively`, which will open a browser window to prompt the user for their login credentials.

``` bash
az login
```

In case you have access to multiple Azure Subscriptions, set the account using the CLI, as described below:

``` bash
az account set -s <Subscription name or Id>
```
## Create AKS Cluster

You can ignore this step if you already have a working AKS cluster.

From your terminal, run the commands below to create a AKS cluster. Note that you need to create a new resource group to host the AKS resource (already covered by the commands).

``` bash
export AKS_RESOURCE_GROUP="<Your Resource Group Name>"
export AKS_CLUSTER="initium-test-aks-cluster" # Set the name of the cluster as you require

# Create Log Analytics Workspace
export AKS_MONITORING_LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace create \
    --resource-group ${AKS_RESOURCE_GROUP} \
    --workspace-name initium-test-aks-workspace \
    --query id \
    -o tsv)

# Create AKS Cluster
az aks create --resource-group ${AKS_RESOURCE_GROUP} \
            --name ${AKS_CLUSTER} \
            --enable-managed-identity \
            --generate-ssh-keys \
            --admin-username aksnodeadmin \
            --node-count 1 \
            --enable-cluster-autoscaler \
            --min-count 1 \
            --max-count 2 \
            --network-plugin kubenet \
            --node-vm-size Standard_DS3 \
            --nodepool-labels nodepool-type=system nodepoolos=linux app=system-apps \
            --nodepool-name systempool \
            --nodepool-tags nodepool-type=system nodepoolos=linux app=system-apps \
            --enable-addons monitoring \
            --workspace-resource-id ${AKS_MONITORING_LOG_ANALYTICS_WORKSPACE_ID} \
            --network-policy calico \
            --vm-set-type VirtualMachineScaleSets \
            --kubernetes-version 1.26.6

# Get Kubernetes credentials
az aks get-credentials --name ${AKS_CLUSTER}  --resource-group ${AKS_RESOURCE_GROUP} 

```
We recommend you to use `Standard_DS3` as the VM Node size as the Initium workloads need some memory (around 14 GiB)

## Initium Platform Setup

### Clone the platform repository

```bash
git clone https://github.com/nearform/initium-platform.git
```

### Install the required tooling

```bash
cd initium-platform
make asdf_install
```

### Login to Azure CLI 

Login using CLI as from above from the root of the `initium-platform` repo 

>Note that at this point you would have a AKS Cluster ready to use.

### Check Cluster access

``` bash

export AKS_RESOURCE_GROUP="<Update your resource group>"
export AKS_CLUSTER="initium-test-aks-cluster"

# Configure Credentials
az aks get-credentials --name ${AKS_CLUSTER}  --resource-group ${AKS_RESOURCE_GROUP} 

# List Nodes
kubectl get nodes

# Cluster Info
kubectl cluster-info

```

### Install ArgoCD
From the root of the `initium-platform` repo run below command

```bash
make argocd
```

### Install ArgoCD Apps

Apply the `initium-platform` app-of-apps.yaml manifest
- Check the [initium-platform releases page](https://github.com/nearform/initium-platform/releases) for the file
- Apply it with
```bash
kubectl apply -f app-of-apps.yaml
```

### Access ArgoCD and wait for the services to go green
- If you installed ArgoCD using `initium-platform`, you should be able to create a port forward to the ArgoCD service.
    ```bash
    kubectl port-forward -n argocd svc/argocd-server 8080:80
    ```
- then you retrieve the admin credentials with
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
- and access it on http://localhost:8080

## Setup Initium CLI.
Ignore these steps in case you already have Initium CLI installed.

- Download the lastest release of the CLI for your operating system [here](https://github.com/nearform/initium-cli/releases) and add it to your PATH.
- Alternatively you can build the CLI from source refer [repo](https://github.com/nearform/initium-cli)

## Deploy demo app via github actions on PR

1. Fork the Initium [NodeJS demo app](https://github.com/nearform/initium-nodejs-demo-app)

> Remember to set the GitHub Actions workflow permissions to "read and write" [here](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#configuring-the-default-github_token-permissions)

2. Setup environment varibale to hold the cluster credentials
    - remember to replace `<YOUR_CLUSTER_NAME>` with your cluster name in below commands

```bash
    initium init service-account | kubectl apply -f -

    export INITIUM_LB_ENDPOINT="$(kubectl get service -n istio-ingress istio-ingressgateway -o go-template='{{(index .status.loadBalancer.ingress 0).ip}}'):80"

    export INITIUM_CLUSTER_ENDPOINT=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "<YOUR CLUSTER NAME>")].cluster.server}')

    export INITIUM_CLUSTER_TOKEN=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.token}" | base64 -d)

    export INITIUM_CLUSTER_CA_CERT=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.ca\.crt}" | base64 -d)
```

3. [Create the following secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in your forked repo

    - CLUSTER_CA_CERT: `echo $INITIUM_CLUSTER_CA_CERT`
    - CLUSTER_TOKEN: `echo $INITIUM_CLUSTER_TOKEN`
    - CLUSTER_ENDPOINT: use the output of `echo $INITIUM_CLUSTER_ENDPOINT` in the format `ADDRESS:PORT`

4. Initialize the initium config and actions in a new branch of the repo you forked

```bash
cd initium-nodejs-demo-app
git checkout -b initium-test
initium init config --persist
initium init github
```

5. Commit the changes and open a PR

6. Wait for the action to finish running and check the logs for the application endpoint

If you followed the guide, the endpoint should look like the following

```bash
curl -H "Host: initium-nodejs-demo-app.initium-test.example.com" $INITIUM_LB_ENDPOINT
```

And the call should return:

```
Hello, World!
```

7. If you merge the PR (DO NOT DELETE THE BRANCH RIGHT AWAY!!!), the service will be removed and a new one will be created for the main branch.

```
curl -H "Host: initium-nodejs-demo-app.main.example.com" $INITIUM_LB_ENDPOINT
```

8. 🚀

