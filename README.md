# freecinc-helm

A [Helm](https://helm.sh/) chart for deploying the FreeCinc service.

## Requirements

* A Kubernetes cluster, all set up and available to `kubectl`
* Tiller installed in that cluster
* Helm client

## Installation

The Helm run will create a "release" from the `./freecinc-com` chart.
Choose an appropriate name for the release (e.g., `staging`), called RELEASE here.

### Secrets

In a working copy of this repository, with Kubernetes set up, first create the required secrets.
Create or gather the following files in a secure location:

* `salt` -- a long, random string used to generate user passwords
* `client.cert.pem` -- from the [taskd setup](https://taskwarrior.org/docs/taskserver/configure.html)
* `client.key.pem`
* `server.cert.pem`
* `server.key.pem`
* `server.crl.pem`
* `ca.cert.pem`
* `ca.key.pem`

Then, create a secret named `RELEASE-secrets`:

```shell
$ kubectl create secret generic dev-secrets \
  --from-file=./salt \
  --from-file=./client.cert.pem \
  --from-file=./client.key.pem \
  --from-file=./server.cert.pem \
  --from-file=./server.key.pem \
  --from-file=./server.crl.pem \
  --from-file=./ca.cert.pem \
  --from-file=./ca.key.pem
```

The Helm chart will refer to this secret, but not modify it.

### Helm Run

To create the release:

```shell
$ helm install --name RELEASE ./freecinc-com
```

To later upgrade it:

```shell
$ helm upgrade RELEASE ./freecinc-com
```

..and to delete it:

```shell
$ helm del RELEASE
```

*NOTE*: To avoid data loss, deleting a release does not delete the underlying persistent volumes.
Delete those manually in the "Storage" tab in GKE or via `kubectl`.

## Operation

This runs a single deployment containing a pod with two containers: one to run [freecinc-taskd](https://github.com/freecinc/freecinc-taskd) and one for the [web frontend](https://github.com/freecinc/freecinc-web).

The containers use two persistent volumes:
 * `pki` contains the generated client certs and keys, and is only used by freecinc-web.
 * `taskddata` contains the taskd database, and is accessed by both containers.

Both containers are also provided with the secrets defined above.
