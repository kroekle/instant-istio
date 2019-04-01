# Instant Istio

This tutorial walks you through doing a basic setup of Istio to an existing Kubernetes cluster.  This is not a production grade setup, but instead is intended to enhance your understanding of Istio basics.

## Target Audience
The target audience for this demo is Engineers/Cluster administrators that want to ehhance their understanding of istio.

## References
[Official Istio Setup Docs](https://istio.io/docs/setup/kubernetes/)


## Prerequisites
This demo assumes that you have a working Kubernetes cluster setup and kubectl command line access to it.  This also assumes you are doing this on linux.

## Install Istio

+ Download the most current version of Isito

`curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.1 sh -`

+ Install Istio CRDs

`cd istio-1.1.1
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done`

+ Install permissive mode resources

`kubectl apply -f install/kubernetes/istio-demo.yaml`

+ Wait until all services/pods are running

`kubectl get svc --namespace=istio-system
kubectl get pods --namespace=istio-system
`

Pods should look something like this when ready:

    grafana-7b46bf6b7c-6844n                  1/1       Running     0          4m
    istio-citadel-5878d994cc-kv5wb            1/1       Running     0          4m
    istio-cleanup-secrets-1.1.1-kzvh2         0/1       Completed   0          4m
    istio-egressgateway-976f94bd-7nrw5        0/1       Running     0          4m
    istio-galley-7855cc97dc-cp8tw             1/1       Running     0          4m
    istio-grafana-post-install-1.1.1-dn5v2    0/1       Completed   0          4m
    istio-ingressgateway-794cfcf8bc-8r5t6     0/1       Running     0          4m
    istio-pilot-746995884c-5qmkh              0/2       Pending     0          4m
    istio-policy-74c95b5657-vk9dd             2/2       Running     3          4m
    istio-security-post-install-1.1.1-rc5wx   0/1       Completed   0          4m
    istio-sidecar-injector-59fc9d6f7d-m6ld2   1/1       Running     0          4m
    istio-telemetry-6c5d7b55bf-wvn55          2/2       Running     0          4m
    istio-tracing-75dd89b8b4-j2xn9            1/1       Running     0          4m
    kiali-5d68f4c676-9mmjb                    1/1       Running     0          4m
    prometheus-89bc5668c-dpd84                1/1       Running     0          4m

##  Enable Istio for namesapce

The most convenient way to use Istio is to use the automatic sidecar injection.  This will add a sidecar image to each pod when it's created or changed in the namespace.  To do that we will need to add a label to the namespace.  We will also want to do an update to the pods that are running in order for the sidecar to be injected.

+ Add label to namespace

`kubectl label namespace <namespace> istio-injection=enabled`

+ Update all the pods

`for i in $(kubectl get deployment --no-headers -o custom-columns=":metadata.name"); do kubectl set env deployment $i --env="force_restart=$(date)"; done`

## Cleanup

It's a good idea to clean up all the Istio artifact as you could be charged (on a clould provider) for the resources they take up.

+ Remove Istio pods

`kubectl delete -f install/kubernetes/istio-demo.yaml`

+ Remove Istio CRDs (optional)

`for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl delete -f $i; done`

+ Remove namespace label

``

+  Restart your pods to remove sidecar
`for i in $(kubectl get deployment --no-headers -o custom-columns=":metadata.name"); do kubectl set env deployment $i --env="force_restart=$(date)"; done`