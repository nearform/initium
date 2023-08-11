# Quick start

In this guide we will see how to start the [Initium Platform](https://github.com/nearform/initium-platform) locally and deploy there from a github action using the [Initium cli](https://github.com/nearform/initium-cli).

## Prerequisites

The Initium Platform uses `kind` so you need at least a way to run Docker.

We use [asdf](https://asdf-vm.com/) to install the remanining tools.

You can use [ngrok](https://ngrok.com/) to expose the platform to the github action.

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

5. Expose the cluster via ngrok

```
ngrok tcp --region us <control-plane-port> 
```

**Note:** you can find the control plane port with `kubectl cluster-info`

## The CLI

1. Download the lastest release of the cli for your operationg system [here](https://github.com/nearform/initium-cli/releases) and added to your PATH.

2. Fork the Initium [nodejs demo app](https://github.com/nearform/initium-nodejs-demo-app)

3. Setup the credentials the cluster credentials

```
initium-cli init service-account | kubectl apply -f -

export INITIUM_LB_ENDPOINT="$(kubectl get service -n istio-ingress istio-ingressgateway -o go-template='{{(index .status.loadBalancer.ingress 0).ip}}'):80"
export INITIUM_CLUSTER_ENDPOINT=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "kind-initium-platform")].cluster.server}')
export INITIUM_CLUSTER_TOKEN=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.token}" | base64 -d)
export INITIUM_CLUSTER_CA_CERT=$(kubectl get secrets initium-cli-token -o jsonpath="{.data.ca\.crt}" | base64 -d)
```

4. Create the following secrets in your forked repo

- CLUSTER_CA_CERT: `echo $INITIUM_CLUSTER_CA_CERT`
- CLUSTER_TOKEN: `echo #INITIUM_CLUSTER_TOKEN`
- CLUSTER_ENDPOINT: use the ngrok endpoint here in the format `#.tcp.ngrok.io:PORT`

5. Clone the forked repo initialize the initium config and actions in a new branch

```
cd initium-nodejs-demo-app
git checkout -b initium-test
initium-cli --app-name "demo-app" init config > .initium.yaml
initium-cli init github
```

6. Commit the changes and open a PR

7. Wait for the PR to finish and check the logs for the application endpoint

If you followed the guide the endpoint should look like the following

```
curl -H "Host: initium-nodejs-demo-app.initium-test.example.com" $KKA_LB_ENDPOING
```

And the call should return:

```
Hello, World!
```

8. If you merge the PR (DO NOT DELETE THE BRANCH) the service will be removed and a new one will be created for the main branch.

```
curl -H "Host: initium-nodejs-demo-app.main.example.com" $KKA_LB_ENDPOING
```

9. 🚀

