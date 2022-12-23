# Seldon Tutorial

If you've tried installing seldon you know the instructions are unhelpful and the docs are poor. This acts as a minimally viable set of instructions in order to get Seldon running locally on a simple example from their example list.


## Updated Guide

1. Install kustomize, kubectl, minikube
2. `git clone` the kubeflow manifests repo [here](https://github.com/kubeflow/manifests.git)
3. `minikube start --driver=docker --nodes=2 --cpus=6 --memory 30000 --kubernetes-version=v1.21.1`  # dedicate more resources to minikube

### Config

`kubectl version`:

Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:52:00Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.3", GitCommit:"434bfd82814af038ad94d62ebe59b133fcb50506", GitTreeState:"clean", BuildDate:"2022-10-12T10:49:09Z", GoVersion:"go1.19.2", Compiler:"gc", Platform:"linux/amd64"}

`minikube version`:

minikube version: v1.28.0
commit: 986b1ebd987211ed16f8cc10aed7d2c42fc8392f

`kustomize version`:

Manually installed from [here](https://github.com/kubernetes-sigs/kustomize/releases/tag/v3.2.0), using binary

Version: {KustomizeVersion:3.2.0 GitCommit:a3103f1e62ddb5b696daa3fd359bb6f2e8333b49 BuildDate:2019-09-18T16:26:36Z GoOs:linux GoArch:amd64}


### Steps

As of Dec 2022, based on the installation instructions mentioned [here](https://github.com/kubeflow/manifests#connect-to-your-kubeflow-cluster), these are the following steps run:

0.
On linux, alias the kustomize version 3.2.0
`alias kustomize=./kustomize_3.2.0_linux_amd64`

1. Make sure in the `manifests` subfolder 

```bash

# Try this set twice, if connection refused error
kustomize build common/cert-manager/cert-manager/base | kubectl apply -f -
kustomize build common/cert-manager/kubeflow-issuer/base | kubectl apply -f -

kustomize build common/istio-1-16/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-16/istio-namespace/base | kubectl apply -f -
kustomize build common/istio-1-16/istio-install/base | kubectl apply -f -

kustomize build common/dex/overlays/istio | kubectl apply -f -

kustomize build common/oidc-authservice/base | kubectl apply -f -

```

At this point, run 

```
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
```

and check pods are all up and running first


```
# Might see some errors here
kustomize build common/knative/knative-serving/overlays/gateways | kubectl apply -f -
kustomize build common/istio-1-16/cluster-local-gateway/base | kubectl apply -f -

kustomize build common/kubeflow-namespace/base | kubectl apply -f -

kustomize build common/kubeflow-roles/base | kubectl apply -f -

kustomize build common/istio-1-16/kubeflow-istio-resources/base | kubectl apply -f -

# Errors here, try twice
kustomize build apps/pipeline/upstream/env/cert-manager/platform-agnostic-multi-user | kubectl apply -f -


kustomize build contrib/kserve/kserve | kubectl apply -f -
kustomize build contrib/kserve/models-web-app/overlays/kubeflow | kubectl apply -f -

kustomize build apps/katib/upstream/installs/katib-with-kubeflow | kubectl apply -f -
kustomize build apps/centraldashboard/upstream/overlays/kserve | kubectl apply -f -

kustomize build apps/admission-webhook/upstream/overlays/cert-manager | kubectl apply -f -


kustomize build apps/jupyter/notebook-controller/upstream/overlays/kubeflow | kubectl apply -f -
kustomize build apps/jupyter/jupyter-web-app/upstream/overlays/istio | kubectl apply -f -

kustomize build apps/volumes-web-app/upstream/overlays/istio | kubectl apply -f -

kustomize build apps/tensorboard/tensorboards-web-app/upstream/overlays/istio | kubectl apply -f -
kustomize build apps/training-operator/upstream/overlays/kubeflow | kubectl apply -f -

kustomize build common/user-namespace/base | kubectl apply -f -
```


Finally,

`kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80` and view on localhost


### Process

1. Have Docker installed locally
2. Install Minikube locally
3. `minikube start --driver=docker`, but note that you might want to make sure you have at least some CPU / free RAM to use to dedicate to running minikube
4. If you haven't already, you'll also need helm and kubectl installed locally
5. Run `kubectl create namespace seldon-system`
6. Beginning to do helm installs; run `helm repo add datawire https://www.getambassador.io`; using Ambassador for ingress vs Istio
7. Might not be needed, but run `helm repo list` and `helm repo update`
8. Install ambassador with helm: `helm install ambassador datawire/ambassador --set image.repository=quay.io/datawire/ambassador --set enableAES=false --set crds.keep=false --namespace seldon-system`
9. Install seldon-core with helm: `helm install seldon-core seldon-core-operator --repo https://storage.googleapis.com/seldon-charts --set ambassador.enabled=true --set usageMetrics.enabled=true --namespace seldon-system`
10. Run this on a separate terminal; here we start port forwarding: `kubectl port-forward $(kubectl get pods -n seldon-system -l app.kubernetes.io/name=ambassador -o jsonpath='{.items[0].metadata.name}') -n seldon-system 8003:8080`. Now we can use localhost:8003 for our model predictions. Note this might take a few seconds to start up
11. Now create the seldon namespace: `kubectl create namespace seldon`
12. Run `kubectl config set-context $(kubectl config current-context) --namespace=seldon`
13. At this point, this is the standard set of instructions for most seldon example deployments. This repo contains the sample sklearn iris model for reference, so we run `kubectl create -f sklearn_iris.yaml`
14. This should launch the pods; you can run `watch -n 1 kubectl get pods` and wait for all the pods to switch to "running" status, or run `kubectl rollout status deploy/$(kubectl get deploy -l seldon-deployment-id=seldon-deployment-example -o jsonpath='{.items[0].metadata.name}')`
15. Voila! If you see the pods running, `curl  -s http://localhost:8003/seldon/seldon/seldon-deployment-example/api/v0.1/predictions -H "Content-Type: application/json" -d '{"data":{"ndarray":[[5.964,4.006,2.081,1.031]]}}` will test that it has spun up correctly!
16. Finally, `minikube delete` after you are done

