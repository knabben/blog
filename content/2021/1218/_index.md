+++
title = "Kube-proxy Windows Kernelspace Mode"
description = "Windows Kernelspace Mode for Services"
date = "2021-12-18"
markup = "mmark"
+++

## Introduction

Sorry not sorry, this is the fourth post about kube-proxy modes, don't worry the evil is in the details. This time these writings will cover
Windows kernelspace mode. This proxy mode is used in Calico and Flannel CNIs and the APIs and golang binding used here (HCN)
are shared on both technologies, so having a good understanding of how the public-facing interaction works and methods
signatures is a good step on enabling the coding on windows container kernel networking capabilities. I will NOT
dive much into the details of how Kube-proxy works AGAIN, the K8S APIs binding on services and endpoints is well covered
on the other posts. Check them out!

## Host Compute Service & Host Compute Network (previous Host Networking Service)

Your first stop should be [here]
(https://kubernetes.io/docs/setup/production-environment/windows/intro-windows-in-kubernetes/). From a brief excerpt:

The Windows HNS (Host Networking Service) and vSwitch implement namespacing and can create virtual NICs as needed for a pod or container.
However, *many configurations such as DNS, routes, and metrics are stored in the Windows registry database* rather than as files inside /etc, 
which is how Linux stores those configurations. The Windows registry for the container is separate from that of the host, so concepts 
like mapping /etc/resolv.conf from the host into a container don't have the same effect they would on Linux. 
*These must be configured using Windows APIs run in the context of that container. Therefore CNI implementations need to call the HNS
 instead of relying on file mappings to pass network details into the pod or container.*

*Overlay networking support in kube-proxy is a beta feature. In addition, it requires KB4482887 to be installed on Windows Server 2019*

![kernel](./images/proxy-kernel.png?width=1024 "kernl")

From a brief introduction of HCN (HNS) use by CNI and the proxy, accordingly to [1]:

Windows container networking is set up similar to Hyper-V virtual machine networking, and in fact it shares many of the internal services, 
especially Host Networking Service (HNS), which cooperates with Host Compute Service (HCS), which manages the containers' life cycles.
When creating a new Docker container, the container receives its own network namespace (compartment) and a Virtual Network Interface Controller (vNIC or in the case of Hyper-V, isolated containers or vmNIC) located in this namespace. The vNIC is then connected to a Hyper-V Virtual Switch (vSwitch), which is also connected to the host default network namespace using the host vNIC.
You can loosely map this construct to the container bridge interface (CBR) in the Linux container world. The vSwitch utilizes Windows Firewall and the Virtual Filtering Platform (VFP) Hyper-V vSwitch extension in order to provide network security, traffic forwarding, VXLAN encapsulation, and load balancing.

This component is crucial for kube-proxy to provide Services' functionalities, and you can think of VFP as iptables from the Linux container world. 
The vSwitch can be internal (not connected to a network adapter on the container host) or external (connected to a network adapter on the container host); it depends on the container network driver. 
In the case of Kubernetes, you will be always using network drivers (L2Bridge, Overlay, Transparent) that create an external vSwitch.

The second source of good information is [2], the video covers well how these mechanism works and a custom implementation using the API.

Let's start with a simple golang to print the supported features available in the server. Using the `github.com/Microsoft/hcsshim/hcn`.

```golang
fmt.Println("+%v", hcn.GetSupportedFeatures())
```

And the good ol` [dev tools](https://github.com/kubernetes-sigs/sig-windows-dev-tools/) for these dangerous maneuvers.

```shell
$ GOOS=windows go build scp winkernel.exe vagrant@10.20.30.11:/tmp; ssh vagrant@10.20.30.11 /tmp/winkernel.exe;
msg="HCN feature check" supportedFeatures="{
    Acl: {
        AclAddressLists:true 
        AclNoHostRulePriority:true 
        AclPortRanges:true 
        AclRuleId:true
    } 
    Api:{
        V1:true 
        V2:true
    }
    RemoteSubnet:true 
    HostRoute:true 
    DSR:false 
    ...
}"
```

In the environment above `SessionAfinity` will NOT work and one last interesting thing to note: overlay (VXLAN) networks on Windows won't
 support dual-stack networking today. The following features should be AVAILABLE for a full experience of the functionalities in the kernel mode.

* IPv6DualStack - hcn.IPV6DualStackSupported()
* SessionAffinity - proxier.supportedFeatures.SessionAffinity {
* DSR (Direct Server Return)[3] - hcn.DSRSupported()
* RemoteSubnet -hcn.RemoteSubnetSupported()

If `Api.V2` is available it's used instead of `Api.V1`.

## Windows kernel mode

Yet from the resource [1] and having another overview from kproxy winkernel:

Kube-proxy is observing Service and Endpoint (endpointslices) objects in order to create **HNS policies** on Windows nodes.
Which are used to implement a Virtual IP address with a value of ClusterIP specified in the Service specification. 

Finally, when a client Pod sends a request to the Virtual IP, it is forwarded using the rules/policies (set up by kube-proxy) to one of the Pods in the Deployment. 
As you can see, kube-proxy is the central component for implementing Services, and in fact it is used for all Service types, apart from ExternalName.

In the case of Kubernetes, CNI plugins are the only way to set up container networking (for Linux, it is possible not to use them). 
They perform the actual communication with HNS and HCS in order to set up the selected networking mode. Kubernetes' networking setup has one significant 
difference when compared to the standard Docker networking setup: the container vNIC is attached to the pod infra container, and the network namespace is 
shared between all containers in the Pod. This is the same concept as for Linux Pods.

## EndpointSlices

From the official K8s documentation [3]. All network endpoints for a Service  were stored in a single Endpoints resource, those resources could get quite large. 
That affected the performance of Kubernetes components (notably the master control plane) and resulted in significant amounts of network traffic 
and processing when Endpoints changed. EndpointSlices help you mitigate those issues as well as provide an extensible 
platform for additional features such as topological routing.
In Kubernetes, an EndpointSlice contains references to a set of network endpoints. The control plane automatically creates
EndpointSlices for any Kubernetes Service that has a selector specified. These EndpointSlices include references to all the Pods
that match the Service selector. 
EndpointSlices group network endpoints together by unique combinations of protocol, port number, and Service name.

*Windows kernel space uses listeners for endpointslices instead of endpoints.* The caching processing stays under `pkg/proxy/endpointslicecache.go`.

Getting the details of the endpointslice used on this post:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: whoami-windows-n2x5d
  namespace: default
addressType: IPv4
endpoints:
- addresses:
  - 100.244.206.69
  targetRef:
    name: whoami-windows-59cd7b4974-pjbq4
- addresses:
  - 100.244.206.68
  targetRef:
    name: whoami-windows-59cd7b4974-6hwph
- addresses:
  - 100.244.206.70
  targetRef:
    name: whoami-windows-59cd7b4974-gr6qg
ports:
- name: ""
  port: 8080
  protocol: TCP
```

### Calico VXLAN and Source VIP

Since Calico is being used and VXLAN overlay is required for Windows on this version, **kube-proxy needs to know the source IP to use for SNAT operations.**
This is set via Kube-proxy *--source-vip* and the Calico_ep IP address.
On *3.19.1* this configuration is `windows-packaging/CalicoWindows/kubernetes/kube-proxy-service.ps1`:

```shell
if ($network.Type -EQ "Overlay") {
    Write-Host "Detected VXLAN network, waiting for Calico host endpoint to be created..."
    while (-Not (Get-HnsEndpoint | ? Name -EQ "Calico_ep")) {
        Start-Sleep 1
    }
    Write-Host "Host endpoint found."
    $sourceVip = (Get-HnsEndpoint | ? Name -EQ "Calico_ep").IpAddress  // 100.244.206.66
    $argList += "--source-vip=$sourceVip"
    $extraFeatures += "WinOverlay=true"    
}
```

It will be used later in the syncProxyRules.

## syncProxyRules deep dive

![sync](./images/sync-proxy.png?width=1024 "sync")

Again this is the core sync proxy rules used by kproxy, it can be divided roughly in 3 parts, and it runs on a scheduled timed goroutine as well as when a 
change occurs in OnServiceSynced and OnEndpointSliceSynced.

The first part represents the bootstrap and is responsible to:

1. Check if the proxier is initialized.
1. Metrics for method latency measurement. 
1. Check if the HNS network was changed or does not exist anymore (this network is created by the CNI). It cleans up ALL policies created.
1. Creates a list of stale UDP services merging endpoints services and service list.

The middle core is responsible to add the corresponding policies for each service. The iteration on each service in the cluster occurs on a 
list of `serviceInfo` cached objects:

```golang
// ServiceInfo contains information and state for a particular proxied service
type serviceInfo struct {
	*proxy.BaseServiceInfo
	targetPort             int
	externalIPs            []*externalIPInfo
	loadBalancerIngressIPs []*loadBalancerIngressInfo
	hnsID                  string
	nodePorthnsID          string
	policyApplied          bool
	remoteEndpoint         *endpointsInfo
	hns                    HostNetworkService
	preserveDIP            bool
	localTrafficDSR        bool
}
```

The first step is to check if the policy is already applied for this service in the `policyApplied` boolean.
If the network is overlay a new REMOTE endpoint is created for the ClusterIP. HNS creates network namespaces
per container endpoints. **A remote endpoint is created for services virtualIP as well on the sync function.**

```golang
serviceVipEndpoint, _ := hns.getEndpointByIpAddress(svcInfo.ClusterIP().String(), hnsNetworkName)
if serviceVipEndpoint == nil {
    newHnsEndpoint, err := hns.createEndpoint(hnsEndpoint, hnsNetworkName)
    svcInfo.remoteEndpoint = newHNSEndpoint
}
```

You can check the existence of this endpoint via powershell utils.

```shell
$ kubectl get svc -o wide
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
whoami-windows   ClusterIP   10.99.234.145    <none>        80/TCP    3h27m   app=whoami-windows
```

Before applying the policy, a new remote endpoint is created for EVERY real endpoint, corresponding to the service.
Getting the existent endpoints for ClusterIP and the container's namespaces from the HNS API:

```shell
$ Get-HNSEndpoint
ID                  : 35F557C2-E194-4424-A021-113DBF8C5F2A
IPAddress           : 10.99.234.145
IsRemoteEndpoint    : True
MacAddress          : 08:00:27:09:de:92
Policies            : {@{PA=10.20.30.11; Type=PA}}
Type                : Overlay

ID                        : 028ABAED-7FF6-4FDC-992C-D3B004936BE0
IPAddress                 : 100.244.206.68
MacAddress                : 0E-2A-64-f4-ce-44
Name                      : 39bd4d8edfb7aa8903b9c342121f17431755266470b98d30b36a5e7d7651d7c6_Calico
Type                      : Overlay

ID                        : E84BAD42-A6EE-40A8-8145-C9F084654774
IPAddress                 : 100.244.206.69
MacAddress                : 0E-2A-64-f4-ce-45
Name                      : bc613b581d44343ec36170adac58050cea252667e9f1077cc41f8105213ad477_Calico
Type                      : Overlay

ID                        : 77C53BAC-A359-45CC-A663-541344EE505A
IPAddress                 : 100.244.206.70
MacAddress                : 0E-2A-64-f4-ce-46
Name                      : 62d2b2d0f5678cd8e77c7ffcf30d0ccfb87cdd759d258084c418ac357a22f760_Calico
Type                      : Overlay
```

Filtering out the logs in the Kube-proxy Load balancer Policy creation for the service we find out:

```shell
I1219 08:24:31.456250    3716 proxier.go:145] Hns loadbalancer policy resource, {
        "ID":"a335d997-4160-44f9-9f72-1087a2cba45e",
        "HostComputeEndpoints":[
            "028abaed-7ff6-4fdc-992c-d3b004936be0",
            "e84bad42-a6ee-40a8-8145-c9f084654774",
            "77c53bac-a359-45cc-a663-541344ee505a"],
        "SourceVIP":"100.244.206.66", "FrontendVIPs":["10.99.234.145"], 
        "PortMappings":[{"Protocol":6,"InternalPort":8080,"ExternalPort":80}],
        "SchemaVersion":{"Major":2,"Minor":0}
    }
I1219 08:24:31.456250    3716 proxier.go:1170] Hns LoadBalancer resource created for cluster ip resources 10.99.234.145, Id [a335d997-4160-44f9-9f72-1087a2cba45e]
```

We can note the list of endpoints a SourceVIP from Calico Remote Endpoint (for SNAT) and FrontendVIPs to the actual Cluster IP.
Find out the HNS Policy list created, note the SourceVIP 100.244.206.66 from the Calico_ep:

```shell
PS C:\k> Get-HnsPolicyList -id A335D997-4160-44F9-9F72-1087A2CBA45E

ActivityId       : FF1908A9-9B2C-4D07-AF3E-865688B344DD
AdditionalParams :
Health           : @{LastErrorCode=0; LastUpdateTime=132844046714528311}
ID               : A335D997-4160-44F9-9F72-1087A2CBA45E
IsApplied        : False
Policies         : {@{ExternalPort=80; InternalPort=8080; Protocol=6; SourceVIP=100.244.206.66; Type=ELB; VIPs=System.Object[]}}        
References       : {/endpoints/028abaed-7ff6-4fdc-992c-d3b004936be0, /endpoints/e84bad42-a6ee-40a8-8145-c9f084654774,
                   /endpoints/77c53bac-a359-45cc-a663-541344ee505a}
Resources        : @{AdditionalParams=; AllocationOrder=1; Allocators=System.Object[]; Health=;
                   ID=FF1908A9-9B2C-4D07-AF3E-865688B344DD; PortOperationTime=0; State=1; SwitchOperationTime=0; VfpOperationTime=0}    
State            : 2
Version          : 38654705666
```

A few other Load balancer policies are created depending of the Service type.

* If nodePort is specified, user should be able to use nodeIP:nodePort to reach the backend endpoints
* Create a Load Balancer Policy for each external IP
* Create a Load Balancer Policy for each loadbalancer ingress

The logic for choosing the SourceVIP in Overlay networks is based on the backend endpoints:

a) Endpoints are any IP's outside the cluster ==> Choose NodeIP as the SourceVIP

b) Endpoints are IP addresses of a remote node => Choose NodeIP as the SourceVIP

c) Everything else (Local POD's, Remote POD's, Node IP of current node) ==> Choose the configured SourceVIP

Finally the last part:

1. Closes the measurement of the function latency.
1. Update services health checks
1. Finish housekeeping - TODO: check if is required to clean up stale services.
1. Remove stale endpoint reference counts

## Monitoring

On Kubernetes 1.24 all endpoints and services metrics are available for windows kernel mode.

![grafana](./images/grafana.png?width=1024 "grafana")

## Conclusion

Definitely the mode is currently the state of art on Windows nodes both for CNIs and Proxies, it's a powerful way to manage
container networks and have been used for a long term. A few other interesting options are happening like eBPF for Windows [4]
or OVS based proxies like Antrea Proxy in the data plane. 

Check the next proxy generation in the newest KPNG project at https://github.com/kubernetes-sigs/kpng
	
## References

[1] https://www.packtpub.com/product/hands-on-kubernetes-on-windows/9781838821562

[2] https://www.youtube.com/watch?v=tTZFoiLObX4

[3] https://techcommunity.microsoft.com/t5/networking-blog/direct-server-return-dsr-in-a-nutshell/ba-p/693710

[4] https://github.com/microsoft/ebpf-for-windows

[5] https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/

