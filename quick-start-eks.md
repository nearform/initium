# Quick start

In this guide we will see how to start the [Initium Platform](https://github.com/nearform/initium-platform) to an EKS cluster and deploy an application to it from a GitHub action using the [Initium CLI](https://github.com/nearform/initium-cli).

## Prerequisites

- A working EKS cluster
  - you can install the `eksctl` binary and create a cluster with it using the tools provided by `asdf` in `initium-platform` as explained below
  - you'll need to give the node role permissions to manage EBS volumes
- The EBS CSI driver installed on the cluster
  - you can follow [these instructions](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) to get it installed
- The cluster's control plane should be publicly exposed so the CLI can reach it
  - remember to check the cluster's security groups

## The Platform

1. Clone the platform repository

```bash
git clone https://github.com/nearform/initium-platform.git
```

2. Install the required tooling

```bash
cd initium-platform
make asdf_install
```

3. Create the EKS cluster (if you don't have one yet)

```bash
AWS_DEFAULT_REGION=<your default region> ~/.asdf/installs/eksctl/<eksctl version installed>/bin/eksctl create cluster
```

4. Install ArgoCD (can be skipped if you already have a cluster and ArgoCD installed in it)

```bash
make argocd
```

5. Apply the `initium-platform` app-of-apps.yaml manifest
    1. Check the [initium-platform releases page](https://github.com/nearform/initium-platform/releases) for the file
    2. Apply it with
    ```bash
    kubectl apply -f app-of-apps.yaml
    ```

6. Access ArgoCD and wait for the services to go green
    1. if you installed ArgoCD using `initium-platform`, you should be able to create a port forwarding to the ArgoCD service
    ```bash
    kubectl port-forward -n argocd svc/argocd-server 8080:80
    ```
    2. then you retrieve the admin credentials with
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
    3. and access it on http://localhost:8080

## The CLI

1. Download the lastest release of the CLI for your operating system [here](https://github.com/nearform/initium-cli/releases) and add it to your PATH.

2. Fork the Initium [NodeJS demo app](https://github.com/nearform/initium-nodejs-demo-app)
    1. Remember to set the GitHub Actions workflow permissions to "read and write" [here](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#configuring-the-default-github_token-permissions)

3. Setup the cluster credentials
    1. remember to replace `<YOUR_CLUSTER_NAME>` with your cluster name

```
initium init service-account | kubectl apply -f -

export INITIUM_LB_ENDPOINT="$(kubectl get service -n istio-ingress istio-ingressgateway -o go-template='{{(index .status.loadBalancer.ingress 0).hostname}}'):80"
export INITIUM_CLUSTER_ENDPOINT=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "<YOUR CLUSTER NAME>")].cluster.server}')
export INITIUM_CLUSTER_TOKEN=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.token}" | base64 -d)
export INITIUM_CLUSTER_CA_CERT=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.ca\.crt}" | base64 -d)
```

4. [Create the following secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in your forked repo

- CLUSTER_CA_CERT: `echo $INITIUM_CLUSTER_CA_CERT`
- CLUSTER_TOKEN: `echo $INITIUM_CLUSTER_TOKEN`
- CLUSTER_ENDPOINT: use the output of `echo $INITIUM_CLUSTER_ENDPOINT` in the format `ADDRESS:PORT`

5. Initialize the initium config and actions in a new branch of the repo you forked

```
cd initium-nodejs-demo-app
git checkout -b initium-test
initium init config --persist
initium init github
```

6. Commit the changes and open a PR

7. Wait for the action to finish running and check the logs for the application endpoint

If you followed the guide, the endpoint should look like the following

```
curl -H "Host: initium-nodejs-demo-app.initium-test.example.com" $INITIUM_LB_ENDPOINT
```

And the call should return:

```
Hello, World!
```

8. If you merge the PR (DO NOT DELETE THE BRANCH RIGHT AWAY!!!), the service will be removed and a new one will be created for the main branch.

```
curl -H "Host: initium-nodejs-demo-app.main.example.com" $INITIUM_LB_ENDPOINT
```

9. 🚀

