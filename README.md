# freecinc-helm

A [Helm](https://helm.sh/) chart for deploying the FreeCinc service.

## Requirements

* A Kubernetes cluster, all set up and available to `kubectl`
* Tiller installed in that cluster
* Helm client

## Installation

In a working copy of this repository, run

```shell
$ helm install --name <your-release-name> ./freecinc-com
```
