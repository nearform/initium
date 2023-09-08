# Quick start

In this guide we will see how to start the [Initium Platform](https://github.com/nearform/initium-platform) on a GKE cluster and deploy an application to it from a GitHub action using the [Initium CLI](https://github.com/nearform/initium-cli).

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

3. Create a standard **(non-autopilot)** GKE cluster (**If there is not a GKE cluster already in place**)
    1. `gcloud` binary should be already in place using `asdf` from before 
    2. Install the `gke-gcloud-auth-plugin` using the following depending on your system:
  
        Recommended way of plugin installation for Windows & OS X:
        ```bash
        gcloud components install gke-gcloud-auth-plugin
        ```
        Recommended way of pluigin installation for Linux:
        - [DEB based systems](https://cloud.google.com/sdk/docs/install#deb)
        - [RPM based systems](https://cloud.google.com/sdk/docs/install#rpm)
        - [Other Linux distros](https://cloud.google.com/sdk/docs/install#linux)

    3. Login to GKE:

        ```bash
        gcloud auth login
        ```

    4. Create GKE cluster
        Replace "YOUR DEFAULT ZONE" and "COMMA-SEPARATED LIST OF NODE ZONES" with your default GCP region

        ```bash
        gcloud container clusters create initium-cluster \
            --zone <YOUR DEFAULT ZONE> \
            --node-locations <COMMA-SEPARATED LIST OF NODE ZONES>
        ```

4. Validate (if not already) that:
    1. Kubernetes Engine API (container.googleapis.com) is enabled in GCP:
        ```bash
        gcloud services enable containerregistry.googleapis.com container.googleapis.com
        ```
    2. The cluster's control plane is network accessible so the CLI can reach it (through a VPN or public networks) - potentially check the cluster's networking options

**Step 5 is not needed if you already have ArgoCD installed in your cluster.**

5. Install ArgoCD

```bash
make argocd
```

6. Apply the `initium-platform` app-of-apps.yaml manifest
    Check the [initium-platform releases page](https://github.com/nearform/initium-platform/releases) for the right file version & apply it. Replace the release version in the next command in both places, if needed. 
        ```bash
        wget -q https://github.com/nearform/initium-platform/releases/download/v0.1.0/app-of-apps.yaml && sed -i 's/v0.1.0/main/' app-of-apps.yaml
        kubectl apply -f app-of-apps.yaml
        ```

7. Access ArgoCD and wait for the services to go green
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

```
initium init service-account | kubectl apply -f -
```

4. [Create the following secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in your forked repo

    Remember to replace `<YOUR_CLUSTER_NAME>` with your cluster name

    - CLUSTER_CA_CERT: `echo $(kubectl get secrets initium-cli-token -o jsonpath="{.data.ca\.crt}" | base64 -d)`
    - CLUSTER_TOKEN: `echo $(kubectl get secrets initium-cli-token -o jsonpath="{.data.token}" | base64 -d)`
    - CLUSTER_ENDPOINT: `echo $(kubectl config view -o jsonpath='{.clusters[?(@.name == "<YOUR CLUSTER NAME>")].cluster.server}')`

5. Initialize the initium config and actions in a new branch of the repo you forked

    ```
    cd initium-nodejs-demo-app
    git checkout -b initium-test
    initium init config --persist
    initium init github
    ```

6. Commit the changes and open a PR. Please use the following values in order to be aligned with the commands on the next steps:
    - app-name = initium-nodejs-demo-app (part of *.initium.yaml*)
    - branch-name = initium-test

7. Wait for the action to finish running and check the logs for the application endpoint

    Set the following env variable for the next command to work.
    ```
    export INITIUM_LB_ENDPOINT="$(kubectl get service -n istio-ingress istio-ingressgateway -o go-template='{{(index .status.loadBalancer.ingress 0).ip}}'):80"
    ```
    
    If you followed the guide, the endpoint should be reachable using the following:
    
    ```
    curl -H "Host: initium-nodejs-demo-app.initium-test.example.com" $INITIUM_LB_ENDPOINT
    ```
    
    And the call should return:
    
    ```
    Hello, World!
    ```
    
    8. If you merge the PR, the service already created with pull request will be removed and a new one will be created for the main branch following the same naming convention as before:
    
    ```
    curl -H "Host: initium-nodejs-demo-app.main.example.com" $INITIUM_LB_ENDPOINT
    ```

9. 🚀

