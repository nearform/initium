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

```
ngrok tcp --region us $(kbl config view --minify --output jsonpath='{.clusters[0].cluster.server}' | awk -F: '{print $NF}') 
```

**Note:** you can find the control plane port with `kubectl cluster-info`

## The CLI

1. Download the lastest release of the CLI for your operating system [here](https://github.com/nearform/initium-cli/releases) and add it to your PATH.

2. Fork the Initium [NodeJS demo app](https://github.com/nearform/initium-nodejs-demo-app)

3. Setup the cluster credentials

```
initium-cli init service-account | kubectl apply -f -

export INITIUM_CLUSTER_ENDPOINT=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "kind-initium-platform")].cluster.server}')
export INITIUM_CLUSTER_TOKEN=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.token}" | base64 -d)
export INITIUM_CLUSTER_CA_CERT=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.ca\.crt}" | base64 -d)
```

4. [Create the following secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in your forked repo

- CLUSTER_CA_CERT: `echo $INITIUM_CLUSTER_CA_CERT`
- CLUSTER_TOKEN: `echo $INITIUM_CLUSTER_TOKEN`
- CLUSTER_ENDPOINT: use the ngrok endpoint here in the format `#.tcp.ngrok.io:PORT`

5. Initialize the initium config and actions in a new branch of the repo you forked

```
cd initium-nodejs-demo-app
git checkout -b initium-test
initium-cli init config > .initium.yaml
initium-cli init github
```

6. Commit the changes and open a PR. Please use the following values in order to be aligned with the commands on the next steps:
    - app-name = initium-nodejs-demo-app (part of *.initium.yaml*)
    - branch-name = initium-test


7. Wait for the action to finish running and check the logs for the application endpoint

    In order to be able to reach out the application that was just deployed follow the steps below depending on the platform Kubernetes cluster runs on. 

    If docker network is reachable from your terminal use the following parameter to setup LB endpoint:

    ```
    export INITIUM_LB_ENDPOINT="$(kubectl get service -n istio-ingress istio-ingressgateway -o go-template='{{(index .status.loadBalancer.ingress 0).ip}}'):80"
    ```
    
    If docker network is not reachable (i.e. Rancher Desktop on M1 Mac) you just need to use port forwarding with istio & setup LB endpoint as follows:

    ```
    kubectl port-forward service/istio-ingressgateway -n istio-ingress 8080:80
    export INITIUM_LB_ENDPOINT="127.0.0.1:8080"
    ```

    If you followed the guide, the endpoint should look like the following:

    ```
    curl -H "Host: initium-nodejs-demo-app.initium-test.example.com" $INITIUM_LB_ENDPOINT
    ```

    And the call should return:

    ```
    Hello, World!
    ```

8. If you merge the PR, the service created on pull request will be removed and a new one will be created for the main branch following the same naming convention as before:

```
curl -H "Host: initium-nodejs-demo-app.main.example.com" $INITIUM_LB_ENDPOINT
```

9. 🚀

