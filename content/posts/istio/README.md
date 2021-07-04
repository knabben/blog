+++
title = "Istio drill down - Part I"
description = "a few notes of Istio tutorials on a boring snowy evening"
date = "2021-03-06"
markup = "mmark"
+++


# Introduction

This post is a review and notes of some Istio tutorials around with the latest version, 
again the base cluster is created by kind and is being used as a CNI.

{{<highlight shell>}} 
NAMESPACE            NAME                                           READY   STATUS    RESTARTS   AGE
kube-system          calico-kube-controllers-5c6f6b67db-8rx2s       1/1     Running   0          4h26m
kube-system          calico-node-9fccq                              1/1     Running   0          4h26m
kube-system          calico-node-b77sb                              1/1     Running   0          4h26m
kube-system          calico-node-jf4z2                              1/1     Running   0          4h26m
kube-system          calico-node-sl9qd                              1/1     Running   0          4h26m
kube-system          coredns-74ff55c5b-dbwpl                        1/1     Running   0          4h27m
kube-system          coredns-74ff55c5b-gpvz7                        1/1     Running   0          4h27m
kube-system          etcd-calico-control-plane                      1/1     Running   0          4h27m
kube-system          kube-apiserver-calico-control-plane            1/1     Running   0          4h27m
kube-system          kube-controller-manager-calico-control-plane   1/1     Running   0          4h27m
kube-system          kube-proxy-cc8sr                               1/1     Running   0          4h27m
kube-system          kube-proxy-gbkl6                               1/1     Running   0          4h27m
kube-system          kube-proxy-j2blv                               1/1     Running   0          4h27m
kube-system          kube-proxy-jp9xm                               1/1     Running   0          4h27m
kube-system          kube-scheduler-calico-control-plane            1/1     Running   0          4h27m
local-path-storage   local-path-provisioner-78776bfc44-m7kpq        1/1     Running   0          4h27m
{{</highlight>}} 


### Installation

This workshop one is made for GCP, but we are using a local cluster with Kind, Bypass old installation methods,
and use the latest installation on v1.21.1. lets use the edges ones:

{{<highlight shell>}} 
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.10.2

# export PATH=${PWD}/bin:$PATH

❯ istioctl version                                                                                                                                                                               [12:27:02]
client version: 1.10.2
control plane version: 1.10.2
data plane version: 1.10.2 (8 proxies)

istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
{{</highlight>}} 

The interesting part is that istiod now became a daemon instead of having the components split as
microservices, so the pods looks cleaner.

More details here:

https://blog.christianposta.com/microservices/istio-as-an-example-of-when-not-to-do-microservices/

{{<highlight shell>}}
> kubectl -n istio-system get pods

istio-system         istio-egressgateway-bd477794-thpxf             1/1     Running   1          16h
istio-system         istio-ingressgateway-79df7c789f-5pp4v          1/1     Running   1          16h
istio-system         istiod-6dc55bbdd-52gjt                         1/1     Running   1          16h
{{</highlight>}} 

Verify the installation: 

{{<highlight shell>}} 
istioctl verify-install
1 Istio control planes detected, checking --revision "default" only
✔ ClusterRole: istiod-istio-system.istio-system checked successfully
✔ ClusterRole: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-istio-system.istio-system checked successfully
✔ Role: istiod-istio-system.istio-system checked successfully
✔ RoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istio-reader-service-account.istio-system checked successfully
✔ ServiceAccount: istiod-service-account.istio-system checked successfully
✔ ValidatingWebhookConfiguration: istiod-istio-system.istio-system checked successfully
✔ CustomResourceDefinition: destinationrules.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: envoyfilters.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: gateways.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: serviceentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: sidecars.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: virtualservices.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadgroups.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: authorizationpolicies.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: peerauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: requestauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: istiooperators.install.istio.io.istio-system checked successfully
✔ ConfigMap: istio.istio-system checked successfully
✔ Deployment: istiod.istio-system checked successfully
✔ ConfigMap: istio-sidecar-injector.istio-system checked successfully
✔ MutatingWebhookConfiguration: istio-sidecar-injector.istio-system checked successfully
✔ PodDisruptionBudget: istiod.istio-system checked successfully
✔ Service: istiod.istio-system checked successfully
✔ EnvoyFilter: metadata-exchange-1.8.istio-system checked successfully
✔ EnvoyFilter: tcp-metadata-exchange-1.8.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.8.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.8.istio-system checked successfully
✔ EnvoyFilter: metadata-exchange-1.9.istio-system checked successfully
✔ EnvoyFilter: tcp-metadata-exchange-1.9.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.9.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.9.istio-system checked successfully
✔ Deployment: istio-ingressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-ingressgateway.istio-system checked successfully
✔ Role: istio-ingressgateway-sds.istio-system checked successfully
✔ RoleBinding: istio-ingressgateway-sds.istio-system checked successfully
✔ Service: istio-ingressgateway.istio-system checked successfully
✔ ServiceAccount: istio-ingressgateway-service-account.istio-system checked successfully
✔ Deployment: istio-egressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-egressgateway.istio-system checked successfully
✔ Role: istio-egressgateway-sds.istio-system checked successfully
✔ RoleBinding: istio-egressgateway-sds.istio-system checked successfully
✔ Service: istio-egressgateway.istio-system checked successfully
✔ ServiceAccount: istio-egressgateway-service-account.istio-system checked successfully
Checked 12 custom resource definitions
Checked 3 Istio Deployments
✔ Istio is installed and verified successfully
{{</highlight>}} 

## Components

The istiod pod in the `istio-system` is the control plane. Some interesting points about the Envoy xDS API,
it will make the data plane very customizable: 

* Listener Discovery Service - LDS
* Endpoint Discovery Servier - EDS
* Route Discovery Service - RDS

The Identity Management is made using the SPIFFE specification (spiffe.org).

Using `istioctl kube-inject` to inject a sidecar on your Pods this will bring an Envoy on your
pod and it will make your data plane, you can check the sidecar inspecting the element:

{{<highlight shell>}} 
 containers:
 - args:
    - proxy
    - sidecar
    - --domain
    - $(POD_NAMESPACE).svc.cluster.local
    - --serviceCluster
    - catalog.$(POD_NAMESPACE)
    - --proxyLogLevel=warning
    - --proxyComponentLogLevel=misc:error
    - --log_output_level=default:info
    - --concurrency
    - "2"
    env:
    ...
{{</highlight>}}

It's possible to use a label istio-injection=enabled in your namespace, and all pods will
be automatically injected.

{{<highlight shell>}}
> kubectl label namespace default istio-injection=enabled
{{</highlight>}}

## Exposing the Ingress

Testing the Ingress gateway can be made via the LoadBalancer port or even the NodePort
used by the host, an example of the usage in the local host:

{{<highlight shell>}}
❯ kubectl get nodes -o wide
NAME                 STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane,master   19m   v1.21.1   172.18.0.2    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2

kubectl get svc istio-ingressgateway -n istio-system

NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.96.163.153   <pending>     15021:31061/TCP,80:30261/TCP,443:31958/TCP,31400:30417/TCP,15443:32533/TCP   15m

❯ curl http://172.18.0.2:30261 -v                                                                                                                                                                [12:35:00]
*   Trying 172.18.0.2:30261...
* TCP_NODELAY set
* Connected to 172.18.0.2 (172.18.0.2) port 30261 (#0)
> GET / HTTP/1.1
> Host: 172.18.0.2:30261
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
  < HTTP/1.1 404 Not Found
  < date: Sun, 04 Jul 2021 15:35:03 GMT
  < server: istio-envoy
  < content-length: 0
  <
* Connection #0 to host 172.18.0.2 left intact
{{</highlight>}}

## Example workload

{{<highlight shell>}} 
> kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
> kubectl get services
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
apigateway    ClusterIP   10.96.232.173   <none>        80/TCP     81m
catalog       ClusterIP   10.96.5.33      <none>        80/TCP     86m
details       ClusterIP   10.96.161.42    <none>        9080/TCP   3m52s
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    26h
productpage   ClusterIP   10.96.2.169     <none>        9080/TCP   3m52s
ratings       ClusterIP   10.96.28.162    <none>        9080/TCP   3m52s
reviews       ClusterIP   10.96.174.147   <none>        9080/TCP   3m52s 
{{</highlight>}} 

Add a gateway and the virtualservice

{{<highlight shell>}} 
> kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
{{</highlight>}} 

Seeing the objects created in the cluster, the Gateway informs the port and protocol and the virtual service will map
the route to the created gateway

{{<highlight yaml>}} 
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
{{</highlight>}} 

Seeing the gateways and services in the system

{{<highlight shell>}} 
$ kubectl get gateways
NAME                 AGE
bookinfo-gateway     17s
$ kubectl get services
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
apigateway    ClusterIP   10.96.232.173   <none>        80/TCP     88m
catalog       ClusterIP   10.96.5.33      <none>        80/TCP     93m
details       ClusterIP   10.96.161.42    <none>        9080/TCP   11m
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    26h
productpage   ClusterIP   10.96.2.169     <none>        9080/TCP   11m
ratings       ClusterIP   10.96.28.162    <none>        9080/TCP   11m
reviews       ClusterIP   10.96.174.147   <none>        9080/TCP   11m
{{</highlight>}}

## visualization

Kali is available to watch the traffic and mapping the services.

{{< img resizedURL="/posts/istio/kali.png" originalURL="./kali.png" style="width: 100%; " >}}

## Next posts

Other explorations on istio on next posts:

- Networking and traffic management - client side load balance, service discovery, circuit breaking, bulk heading, timeout, retry, mirroring..
- Security - SPIFFE, mTLS
