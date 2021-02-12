# Seldon Tutorial

If you've tried installing seldon you know the instructions are unhelpful and the docs are poor. This acts as a minimally viable set of instructions in order to get Seldon running locally on a simple example from their example list.

### Process

1. Have Docker installed locally
2. Install Minikube locally
3. `minikube start`, but note that you might want to make sure you have at least some CPU / free RAM to use to dedicate to running minikube
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

