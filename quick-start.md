# Quick start

In this guide we will see how to start the [Initium Platform](https://github.com/nearform/initium-platform) locally and deploy an application to it from a github action using the [Initium CLI](https://github.com/nearform/initium-cli).

## Prerequisites

The Initium Platform uses `kind` so you need at least a way to run Docker.

We use [asdf](https://asdf-vm.com/) to install the remaining tools.

You can use [ngrok](https://ngrok.com/) to expose the platform to the GitHub action.

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

3. Start the platform

    ```bash
    make
    ```

4. Wait for the initum-platform to be ready in the Tilt interface.

    You can check Tilt UI by using [this](http://localhost:10350) URL.


5. Expose the cluster via ngrok

    ```bash
    ngrok tcp --region us $(kubectl config view -o jsonpath='{.clusters[?(@.name == "kind-initium-platform")].cluster.server}' | awk -F: '{print $NF}') 
    ```

## The CLI

1. Download the lastest release of the CLI for your operating system [here](https://github.com/nearform/initium-cli/releases) and add it to your PATH.

2. Fork and Clone the Initium [NodeJS demo app](https://github.com/nearform/initium-nodejs-demo-app)

3. Setup the cluster credentials

    Create service account:
    ```bash
    initium-cli init service-account | kubectl apply -f -
    ```

    Get the secrets:
    ```bash
    export INITIUM_CLUSTER_ENDPOINT=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "kind-initium-platform")].cluster.server}')
    export INITIUM_CLUSTER_TOKEN=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.token}" | base64 -d)
    export INITIUM_CLUSTER_CA_CERT=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.ca\.crt}" | base64 -d)
    ```

4. [Create the following secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in your forked repo

    #### CLUSTER_CA_CERT
    ```bash
    echo $INITIUM_CLUSTER_CA_CERT
    ```
    #### CLUSTER_TOKEN
    ```bash
    echo $INITIUM_CLUSTER_TOKEN
    ```
    #### CLUSTER_ENDPOINT
    ngrok endpoint in the format `#.tcp.ngrok.io:PORT`

3. Expose the platform load balancer

    ```
    export INITIUM_LB_ENDPOINT="$(kubectl get service -n istio-ingress istio-ingressgateway -o go-template='{{(index .status.loadBalancer.ingress 0).ip}}'):80"
    ```

    **Alternative: we noticed that in some setups the docker network is not reachable, in that scenario you can expose the service with:**

    ```bash
    kubectl port-forward service/istio-ingressgateway -n istio-ingress 8080:80 &
    export INITIUM_LB_ENDPOINT="127.0.0.1:8080"
    ```


1. Initialize the initium config and actions in a new branch of the repo you forked

    ```bash
    cd initium-nodejs-demo-app && \
    git checkout -b test && \
    initium-cli init config --persist && \
    initium-cli init github
    ```

2. Commit and push the changes and open a PR

    ```bash
    git add .
    git commit -m "Initium start"
    git push
    ```

    **NOTE**: Be sure you open a PR on your own repo


8. Curl the application
   
    If you followed the guide, the application should be reachable using the following.

    ```
    curl -H "Host: initium-nodejs-demo-app.initium-test.example.com" $INITIUM_LB_ENDPOINT
    ```

    *Note: you can find the endpoint in the output of the github action*

9.  Merge the PR

    This will delete the service deployed by the PR and create a new one from main

    ```
    curl -H "Host: initium-nodejs-demo-app.main.example.com" $INITIUM_LB_ENDPOINT
    ```

10. You can repeat the last 4 steps and have an ephemeral environment for each PR 🚀

