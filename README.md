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

```
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.2 sh -
```

+ Install Istio CRDs

```
cd istio-1.1.2
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```

+ Install permissive mode resources

```
kubectl apply -f install/kubernetes/istio-demo.yaml
```

+ Wait until all services/pods are running

```
kubectl get svc --namespace=istio-system
kubectl get pods --namespace=istio-system
```

Pods should look something like this when ready:
```
NAME                                      READY     STATUS      RESTARTS   AGE
grafana-7b46bf6b7c-wm2lk                  1/1       Running     0          4m
istio-citadel-5878d994cc-vxdth            1/1       Running     0          4m
istio-cleanup-secrets-1.1.1-grqr5         0/1       Completed   0          4m
istio-egressgateway-976f94bd-pvtts        1/1       Running     0          4m
istio-galley-7855cc97dc-r55gm             1/1       Running     0          4m
istio-grafana-post-install-1.1.1-p69t8    0/1       Completed   0          4m
istio-ingressgateway-794cfcf8bc-522bj     1/1       Running     0          4m
istio-pilot-746995884c-nds8h              2/2       Running     0          4m
istio-policy-74c95b5657-bhxpd             2/2       Running     4          4m
istio-security-post-install-1.1.1-7mxr5   0/1       Completed   0          4m
istio-sidecar-injector-59fc9d6f7d-rnrmt   1/1       Running     0          4m
istio-telemetry-6c5d7b55bf-bp6vw          2/2       Running     4          4m
istio-tracing-75dd89b8b4-thhwl            1/1       Running     0          4m
kiali-5d68f4c676-fw2l4                    1/1       Running     0          4m
prometheus-89bc5668c-85fvf                1/1       Running     0          4m
```

##  Enable Istio for namesapce

The most convenient way to use Istio is to use the automatic sidecar injection.  This will add a sidecar image to each pod when it's created or changed in the namespace.  To do that we will need to add a label to the namespace.  We will also want to do an update to the pods that are running in order for the sidecar to be injected.

+ Add label to namespace

```
kubectl label namespace <namespace> istio-injection=enabled
```

+ Update all the pods

```
for i in $(kubectl get deployment --no-headers -o custom-columns=":metadata.name"); do kubectl set env deployment $i --env="force_restart=$(date)"; done
```

## Visualizing 

The default setup will install Kiali for visualizing the running services.  Use the following to view Kiali locally.  Default user:pass is admin:admin.

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
http://localhost:20001/kiali/console
```

## Metrics

The default setup will install Grafana for dislaying metrics.  Use the following to view Grafana locally.  No login will be required by default.

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
http://localhost:3000/dashboard/db/istio-mesh-dashboard
```

## Retries

You can configure automatic retries on a service with the following config.  The retryOn attribute can be comma delimited of values found [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/router_filter#x-envoy-retry-on) 

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: <name>
spec:
  hosts:
    - <service name>
  http:
  - route:
    - destination:
        host: <service name>
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx 
EOF
```

## Controlling Ingress

Istio has a concept called an IngressGateway.  This is different then a standard Kubernetes ingress, in fact, if you currently use a k8s ingress, this (as well as a change to the VirtualService) will replace it.  

The following setups a new Gateway, which will add configuration to the default istio ingressgateway.

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: <gateway name>
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

The following is what your new virtualservice looks like:

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: <name>
spec:
  hosts:
    - "*"
  gateways:
  - <gateway name>
  http:
  - match:
    - uri:
        prefix: /<path>
    route:
    - destination:
        host: <service name>
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx 
EOF
```

## Injecting Faults

You can configure the VirtualService to inject faults into your system (chaos testing).  The following example will add delays and faults to 20% of the request coming in

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: <name>
spec:
  hosts:
    - <name>
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percent: 20
      abort:
        httpStatus: 409
        percent: 20
    route:
    - destination:
        host: <name>
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
EOF
```

## Whitelisting/Blacklisting

From within Istio you can whitlist/blacklist requests.  Not only can you do this based on IP address or attributes.  This example will walk you through a simple whitelist of an ip address.

The first resource is a list checker where we define the ip address and that we want to whitelist it

```
kubectl apply -f - <<EOF
apiVersion: config.istio.io/v1alpha2
kind: listchecker
metadata:
  name: whitelistip
spec:
  overrides: ["<ip address>/32"]  # overrides provide a static list
  blacklist: false
  entryType: IP_ADDRESSES
EOF

```

The second resource is a list entry that gets the source ip of the request

```
kubectl apply -f - <<EOF
apiVersion: config.istio.io/v1alpha2
kind: listentry
metadata:
  name: sourceip
spec:
  value: source.ip | ip("0.0.0.0")
EOF
```

The thrid resource is the rule that references the previous two resources to create the whitelist 

```
kubectl apply -f - <<EOF
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: checkip
spec:
  match: source.labels["istio"] == "ingressgateway"
  actions:
  - handler: whitelistip.listchecker
    instances:
    - sourceip.listentry
EOF
```


## Cleanup

It's a good idea to clean up all the Istio artifact as you could be charged (on a clould provider) for the resources they take up.

+ Remove Istio pods

```
kubectl delete -f install/kubernetes/istio-demo.yaml
```

+ Remove Istio CRDs (optional)

```
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl delete -f $i; done
```

+ Remove namespace label

```
kubectl label namespace default istio-injection-
```

+  Restart your pods to remove sidecar
```
for i in $(kubectl get deployment --no-headers -o custom-columns=":metadata.name"); do kubectl set env deployment $i --env="force_restart=$(date)"; done
```
