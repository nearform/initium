# Quick start

In this guide we will see how to start the [Initium Platform](https://github.com/nearform/initium-platform) to an EKS cluster and deploy an application to it from a GitHub action using the [Initium CLI](https://github.com/nearform/initium-cli).

## Tools Required

You will need the tools below in order to follow this guide. If you don't have them installed in your machine, that's fine. They can be installed using `asdf_install` from the `initium-platform` repository, as shown in the next steps.

- eksctl
- awscli
- [Initium Platform](https://github.com/nearform/initium-platform)
- [Initium CLI](https://github.com/nearform/initium-cli)


## EKS Cluster Requirements

If you do not have a working cluster that meets the requirements below, then proceed with the instructions from this section.

- EKS Cluster in ready state, with at least 2 instances of type `m5_large` or bigger. 
- EBS CSI Driver should be installed as Grafana Loki will use an EBS disk.
- The cluster's control plane should be publicly exposed so the CLI can reach it
  - remember to check the cluster's control plane security group for public access on port 443

We are going to use `eksctl` and `aws` to provision the EKS cluster.

### Authenticate with AWS

Get the AWS credentials for programmatic access and run the commands below. Then, enter the credentials when prompted. Refer to the [AWS docs](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html) for other methods to authenticate with the AWS CLI.

```bash
$ aws configure
  AWS Access Key ID [None]: <Your AWS Access Key ID>
  AWS Secret Access Key [None]: <Your AWS Secret Access Key>
  Default region name [None]: <Your AWS Region>
  Default output format [None]: json
$ aws configure set aws_session_token <Your AWS Session Token - if your organization uses them>

```

### Create EKS Cluster

Create cluster using eksctl by running below command from the terminal. Note that this command will provision a Kubernetes cluster with default configuration. It will be a 2 node cluster and by default allows all traffic to the nodes. And set your default region code in the environment variable

```bash
# Set the AWS deployment region
export AWS_DEFAULT_REGION=eu-north-1

# Create EKS Cluster using eksctl cli
eksctl create cluster

# Get Cluster name by running below command and grab the name for next command.
eksctl get clusters

# Set environemt variable for storing EKS Cluster name for further use.
export CLUSTER_NAME=<Your desired Cluster Name>

```

### Create and associate an IAM OIDC Provider with the EKS cluster

This enables us to use AWS IAM Roles for Kubernetes service accounts on our EKS Cluster.


```bash
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_DEFAULT_REGION}  \
    --cluster ${CLUSTER_NAME} \
    --approve
```

### Install EBS CSI driver add-on

The EBS CSI driver is used by the Grafana Loki service and we need to perform the steps below in order to install it to the cluster.

#### Create EBS CSI Driver role

Create an IAM role and attach the required AWS managed policy with the following command. 

```bash
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster ${CLUSTER_NAME} \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

#### Adding the Amazon EBS CSI driver add-on

Run the following command:

```bash
eksctl create addon --name aws-ebs-csi-driver --cluster ${CLUSTER_NAME} --service-account-role-arn arn:aws:iam::567858888620:role/AmazonEKS_EBS_CSI_DriverRole --force

```

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

3. [Create the EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) (if you don't have one yet)

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

8. If you merge the PR, the service will be removed and a new one will be created for the main branch.

```
curl -H "Host: initium-nodejs-demo-app.main.example.com" $INITIUM_LB_ENDPOINT
```

9. 🚀

