# The Initium Project

**Get your code deployed on day zero avoiding vendor lock-in.**

Initium - latin from ineō (“go in, make a start”) +‎ -ium, the former from in (“in, into”) +‎ eō (“go”).

## What

Initium is a project maintained by [NearForm](https://nearform.com), composed by:

### The Initium CLI

A single binary to build and deploy your code.

You can forget about:

- Dockerfile
- Kubernetes manifest
- CI Pipelines

The Initium CLI will manage these for you, including setting up a nice workflow with ephemeral environments for you PRs.

You can find the CLI [here](https://github.com/nearform/initium-cli)

### The Initium Platform

A platform with optimal configuration and test coverage to setup a modern cloud-native platform starting from single machine or a Kubernetes clusters.

The platform offers full:

- **Observability**: Logs, Metrics, Traceses enabled by default thanks to Grafana, Prometheus, Loki, Open-telemetry.

- **Scalability and flexibility**: Test your code under load and enable complex behaviour like blue-green or canary deployments thanks to Knative and Istio

You can find the Platform [here](https://github.com/nearform/initium-platform)

## Why

We created Initium to solve an issue that we and many of the organizations we work with encounter in almost every new project. The issue is that Software and Infrastructure Engineers start writing code almost in parallel, but they get to the first deployable unit at different times. 

Infrastructure Engineers must then wait for the account to be provisioned, access to be provided, and technology to be chosen. Only then can they start testing their Infrastructure as Code, and even here the iteration loop is slower than the software iteration.

Developers, on the other hand, can have a simple ‘Hello World’ server ready in less than 10 minutes. However, if they want to deploy this server then they'll have to wait for the infrastructure to be prepared, get access, understand how to deploy the code, write a Dockerfile, create one or more pipelines and come up with a proper workflow to iterate.

## Documentation

Find all the documentation and getting started guides in the [Initium website](https://initium.nearform.com)
