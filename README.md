# freecinc-helm

A [Helm](https://helm.sh/) chart for deploying the FreeCinc service.

## Apologies

This was an experiment with Helm.
The findings in this experiment: Helm is junk.
It can't actually update resources, and will often "succeed" (meaning fail with the usual errors) without actually updating anything.
It's really no better than just running `kubectl apply -f ..` in a loop -- in fact, it's much worse for making promises to do more and failing.

Kubernetes is really not an appropriate tool for a service of this tiny size.
In fact, this could be deployed easily in Heroku or the like if taskd used a "normal" HTTP termination.
However, it relies on client TLS certificates, in a way that prevents use of proxies or HTTP load balancers.

In order to reduce complexity and limit cost, the current deployment uses a simple TCP load balancer for all traffic, including the freecinc-web traffic.
That precludes use of TLS for that web traffic, meaning that we are serving new keys over HTTP.
That's OK, though -- the entire taskd protocol is insecure.

## Requirements

* A GKE Kubernetes cluster, all set up and available to `kubectl`
    * GKE is used to get a built-in ingress controller and cert management
* A local install of Helm 3

## Installation

The Helm run will create a "release" from the `./freecinc-com` chart.
Choose an appropriate name for the release (e.g., `staging`), called RELEASE here.

You'll also need to supply the hostname for the service (HOSTNAME).

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

**NOTE**: you will want to ensure that the certificates have a lifetime greater
than the default (1 year) as there is no way to rotate them.

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
$ helm install RELEASE --set hostname=HOSTNAME ./freecinc-com
```

To later upgrade it:

```shell
$ helm upgrade RELEASE ./freecinc-com
```

or if that doesn't work (Helm fails a lot)

```shell
$ helm install --set hostname=HOSTNAME --replace --name RELEASE ./freecinc-com
```

If this fails to update a resource, one way to force things is to delete the resource with kubectl and re-run one of the above commands.

To delete the release (*including persistent volumes*, and noting that this is not Helm's strong point):

```shell
$ helm del RELEASE
```

### DNS

Once this is complete and everything has "settled", get your release's IP as described in the notes.
Add that to the DNS for HOSTNAME, and wait for the [managed certificate](https://console.cloud.google.com/net-services/loadbalancing/advanced/sslCertificates/list) to provision.
With that, `https://HOSTNAME` should show the FreeCinc website!

## System Operations

The easiest way to back up the persistent volumes is to use `kubectl exec` on a running pod.
Get the pod name in POD and then

```shell
$ kubectl exec $POD -c freecinc-web --  tar -C / -cvf - /taskddata /pki > backup.tar
```

Similarly, to restore (noting that this will not remove other files that might exist):

```shell
$ kubectl exec -i $POD -c freecinc-web --  tar -C / -xvf - < backup.tar
```

## Design

This runs a single deployment containing a pod with two containers: one to run [freecinc-taskd](https://github.com/freecinc/freecinc-taskd) and one for the [web frontend](https://github.com/freecinc/freecinc-web).

The containers use two persistent volumes:
 * `pki` contains the generated client certs and keys, and is only used by freecinc-web.
 * `taskddata` contains the taskd database, and is accessed by both containers.

Both containers are also provided with the secrets defined above.
