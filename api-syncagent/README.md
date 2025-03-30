# api-syncagent

A demo showcasing the [kcp-dev/api-syncagent](https://github.com/kcp-dev/api-syncagent) with Crossplane.

## Setup

### Prerequisites

* kind
* helm
* dagger
* cloud-provider-kind
* kubectl
* kcp
* jq

### Prepare the Kubernetes cluster

```bash
# we create a kind cluster to serve the WebApp resources
kind create cluster
# we install crossplane and envoy-gateway
helm upgrade --install crossplane https://charts.crossplane.io/stable/crossplane-1.19.1.tgz --namespace crossplane-system --create-namespace
helm upgrade --install envoy-gateway oci://docker.io/envoyproxy/gateway-helm --version 1.3.2 --namespace envoy-gateway-system --create-namespace
# we install the dependencies to serve the WebApp API
kubectl apply -f manifests/provider.yaml
kubectl apply -f manifests/providerconfig.yaml
kubectl apply -f manifests/function.yaml
kubectl apply -f manifests/gateway.yaml
# we generate and install the crossplane manifests for the WebAPP API from [api.cue](./api.cue)
dagger -m github.com/orvis98/api-tool call gen --api-spec ./api.cue | kubectl apply -f - --server-side
# inspect the API
kubectl explain webapps.example.com.spec
KIND:     WebApp
VERSION:  example.com/v1alpha1

RESOURCE: spec <Object>

DESCRIPTION:
     <empty>

FIELDS:
   basicAuth    <Object>
     BasicAuth defines the configuration for the HTTP Basic Authentication.

    ...

   containerPort        <integer> -required-
     Number of HTTP port to expose on the pod's IP address.

   env  <map[string]string>
     Environment variables to set in the container

   image        <string> -required-
     Container image name.

    ...

# configure the WebApp API as a published resource
kubectl apply -f https://raw.githubusercontent.com/kcp-dev/api-syncagent/refs/tags/v0.2.0/deploy/crd/kcp.io/syncagent.kcp.io_publishedresources.yaml
kubectl apply -f manifests/publishedresource.yaml
```

### Prepare the kcp instance

```bash
# start a local kcp instance
kcp start
# create workspaces
alias kcpctl="KUBECONFIG=.kcp/admin.kubeconfig kubectl"
kcpctl ws create org1
kcpctl ws :root:org1
kcpctl ws create --type team team1
# export the API from org1
kcpctl ws :root:org1
kcpctl apply -f manifests/apiexport.yaml
# bind to the API from team1
kcpctl ws :root:org1:team1
kcpctl apply -f manifests/apibinding.yaml
```

### Start the api-syncagent

```bash
# prepare credentials
KUBECONFIG=.kcp/admin.kubeconfig kubectl ws :root:org1
cp .kcp/admin.kubeconfig .kcp/org1.kubeconfig
kind export kubeconfig --kubeconfig kind.kubeconfig
# start a local api-syncagent instance that can talk to both kcp and k8s
api-syncagent --apiexport-ref org1.example.com --kcp-kubeconfig .kcp/org1.kubeconfig --kubeconfig kind.kubeconfig --namespace default
```

## Demo

We need an IP address accessible from the host for the Gateway.

```bash
# running cloud-provider-kind should give the Gateway an address
sudo cloud-provider-kind
export GATEWAY_IP="$(kubectl get gateway webapps -o json | jq '.status.addresses[0].value' --raw-output)"
```

Now we are ready to consume the API from the `team1` Workspace.

```bash
alias kcpctl="KUBECONFIG=.kcp/admin.kubeconfig kubectl"
kcpctl ws :root:org1:team1
# create a WebApp running podinfo
kcpctl apply -f manifests/webapp.yaml 
# extract its hostname
export PODINFO_HOSTNAME="$(kcpctl get webapp podinfo -o json | jq '.status.hostnames[0]' --raw-output)"
# try to get version info without credentials
curl -H "Host: $PODINFO_HOSTNAME" $GATEWAY_IP
User authentication failed. Missing username and password.
# try with invalid credentials
curl -H "Host: $PODINFO_HOSTNAME" -u foo:baz $GATEWAY_IP
User authentication failed. Invalid username/password combination.
# try with valid credentials
curl -H "Host: $PODINFO_HOSTNAME" -u foo:bar $GATEWAY_IP
{
  "hostname": "default-podinfo-9cf5dc6c6-hlcd2",
  "version": "6.8.0",
  "revision": "b3396adb98a6a0f5eeedd1a600beaf5e954a1f28",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.8.0",
  "goos": "linux",
  "goarch": "arm64",
  "runtime": "go1.24.1",
  "num_goroutine": "6",
  "num_cpu": "10"
}
```

How does this look in the Kubernetes cluster?

```bash
# we can see that a composition was created in the Kubernetes cluster with names from kcp
kubectl get xwebapp
NAME                    SYNCED   READY   COMPOSITION                     AGE
default-podinfo-h5rrb   True     False   xwebapps.example.com-v1alpha1   7m10s
# we can see that a new Namespace was created for the kcp Workspace
kubectl get ns
NAME                   STATUS   AGE
crossplane-system      Active   3h37m
default                Active   3h43m
envoy-gateway-system   Active   3h41m
j76ketwmralcrn82       Active   15m
kube-node-lease        Active   3h43m
kube-public            Active   3h43m
kube-system            Active   3h43m
local-path-storage     Active   3h43m
# we can see that a WebApp claim was created in the Namespace
kubectl -n j76ketwmralcrn82 get webapp
NAME              SYNCED   READY   CONNECTION-SECRET   AGE
default-podinfo   True     True                        5m37s
# we can also see the related objects
kubectl get object
NAME                                   KIND             PROVIDERCONFIG   SYNCED   READY   AGE
default-podinfo-h5rrb-deployment       Deployment       default          True     True    7m22s
default-podinfo-h5rrb-httproute        HTTPRoute        default          True     True    7m22s
default-podinfo-h5rrb-secret           Secret           default          True     True    7m22s
default-podinfo-h5rrb-securitypolicy   SecurityPolicy   default          True     True    7m22s
default-podinfo-h5rrb-service          Service          default          True     True    7m22s
# and the actual resources
kubectl -n j76ketwmralcrn82 get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/default-podinfo-9cf5dc6c6-hlcd2   1/1     Running   0          9m4s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/default-podinfo   ClusterIP   10.96.48.164   <none>        80/TCP    9m4s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/default-podinfo   1/1     1            1           9m4s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/default-podinfo-9cf5dc6c6   1         1         1       9m4s
```
