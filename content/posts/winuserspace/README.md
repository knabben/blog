+++
title = "Kube-proxy Winuserspace Mode"
description = "Windows Userspace Mode for Services"
date = "2021-10-17"
markup = "mmark"
+++

## Introduction

This post will introduce and detail the windows userspace mode on kube-proxy, the code walkthrough provided here has 
the goal to clarify the event based architecture of kube-proxy, and bring attention to how this component watches and 
reacts for events happening on Services and Endpoints API objects. This component goal is to proxy and load-balance the traffic
to a binded IP/PORT pair across different backend services. Read more about Kube-Proxy in the official docs [1].

### Windows Isolation Modes

Windows offer two distinct modes of runtime isolation: processes and hyper-v. We are using `process` since Kubernetes
currently does not support running Windows containers with Hyper-V isolation. [2]

According [3], this is the default isolation mode provided for containers on Windows and the architecture of process isolation 
is similar to what you have when running containers on Linux os. All containers use the same shared kernel.
The process isolation providers a lightweight runtime for containers (compared to hyper-v isolation) and offers 
a great density of deployment, better performance and lower spin-up time.

The setup is using the `sig-windows-dev-tools` with the `Antrea CNI on 1.3.0`

### Kube-proxy and Networking model

In short, networking in Kubernetes has the following challenges to overcome:

- Intra-Pod communication between containers: Handled by standard localhost communication.
- Pod-to-Pod communication: Handled by underlying network implementation. 
- Pod-to-Service and External-to-Service communication: Handled by Service API objects, communication is dependent on the underlying network implementation. We will cover this later in this section.
- Automation of networking setup by kubelet when a new Pod is created: Handled by Container Network Interface (CNI) plugins. We will cover this in the next section.

Kube-proxy handles the pod and external to service requests part.

## Kube-Proxy Windows Modes

For Windows, two modes are supported, winuserspace, used by CNIs like Antrea, and deprecated, since it's
the most simple, this will be the initial one and will give the not familiar reader a better overview
of the kube-proxy model.

The second mode is the Windows kernel mode, used by Project Calico, this is more complex, and uses the 
Golang SHIM for HCN (Host Compute Network) service API, and is the recommended mode for Windows. This indeed
is the coolest part and it's coming next, stay tuned.

## Kubernetes and Kube-proxy

{{< img resizedURL="/posts/winuserspace/bootstrap.jpg" originalURL="./bootstrap.jpg" style="width: 100%; " >}}

This initial workflow introduces the initial path of Event handler registering and Service/Endpoint listeners and
watchers, the black box shows the binary called. The initial `cli.Run` is the beginning of `spf13/cobra` 
command integration with Kube-proxy, the function `NewProxyCommand` returns the base `cobra.Command` and the
`Options` object is used to:

1. Complete() - Load the configuration from the file if it exists and set all default parameters.
2. Validate() - Validate the field values based on pre-defined API logic.
3. Run() - Real deal.

Initially the `NewProxyServer` call returns a `proxyServer` object based in the operating system and parameters
passed via args, this will allow the user get the correct mode settings, for Linux, userspace, ipvs and iptables
are available. For Windows as already said, it exists userspace and kernelspace. 

This is set at compilation time using the `_windows` prefix compiling our files selected for this OS. The core and central object
here is the `proxyServer.proxier` one, it is fetch as an instance from `winuserspace.Proxier` and filtered on `server_windows.go:winuserspace.NewProxier(...)`,
in our case.

This main loop on `runLoop` only run the `ProxyServer.Run()` in a goroutine and listen for an errorChannel to finalize the 
infinite loop of the main goroutine.

This bootstrap process is similar to all different services and operating systems. And the `ProxyServer` run do a few interesting things:

* Starts the oomAdjuster
* serveHealthz and serveMetrics
* conntracker (nil for windows)
* Create configs (watchers for services and endpoints/endpointslices)

This is a very important part, since at this point we are going to start the register of the internal
proxies service/endpoint handling functions to Kubernetes objects events. More details in the next section.

```golang
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.Proxier)
	go serviceConfig.Run(wait.NeverStop)

	if endpointsHandler, ok := s.Proxier.(config.EndpointsHandler); ok && !s.UseEndpointSlices {
		endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().V1().Endpoints(), s.ConfigSyncPeriod)
		endpointsConfig.RegisterEventHandler(endpointsHandler)
		go endpointsConfig.Run(wait.NeverStop)
	} 
```

Finally, it starts the SyncLoop of the Proxier in the end of the function:

```golang
	go s.Proxier.SyncLoop()
```

The `SyncLoop` method, it's normally heavily used on other modes, but winuserspace, it's 
only used to clean stale affinity sessions, the ones not used by the TTL setting.

```golang
	if int(time.Since(affinity.lastUsed).Seconds()) >= state.affinity.ttlSeconds {
```

### Services and Endpoints informers

To understand what the functionality is being implemented here. This was extracted from the official docs [4]:

In this (legacy) mode, kube-proxy watches the Kubernetes control plane for the addition and removal of Service and Endpoint objects. 
For each Service it opens a port (randomly chosen) on the local node. Any connections to this "proxy port" are proxied to one of the Service's 
backend Pods (as reported via Endpoints). kube-proxy takes the SessionAffinity setting of the Service into account when deciding which 
backend Pod to use.

Lastly, the user-space proxy installs iptables rules which capture traffic to the Service's clusterIP (which is virtual) and port. The rules redirect that traffic to the proxy port which proxies the backend Pod. For Windows the netns tool is used instead of iptables.

By default, kube-proxy in userspace mode chooses a backend via a round-robin algorithm.

{{< img resizedURL="/posts/winuserspace/listener.png" originalURL="./listener.png" style="width: 180%; " >}}

Starting the fun. From the green boxes, we can see the `proxier` is registered on both `serviceConfig` and `endpointsConfig`, providing
an informer on `Add`, `Update`, `Delete` for both objects. The further connection starts on the pink boxes with `On*Add/Update/Delete`.

#### Services

The more important methods here are the `mergeService` and `unmergeService`, basically they are responsible
to start and remove the listeners and bind/forward the connections to the backends in a round-robin fashion.

```golang
// OnServiceAdd is called whenever creation of new service object
// is observed.
func (proxier *Proxier) OnServiceAdd(service *v1.Service) {
	_ = proxier.mergeService(service)
}

// OnServiceUpdate is called whenever modification of an existing
// service object is observed.
func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
	existingPortPortals := proxier.mergeService(service)
	proxier.unmergeService(oldService, existingPortPortals)
}

// OnServiceDelete is called whenever deletion of an existing service
// object is observed.
func (proxier *Proxier) OnServiceDelete(service *v1.Service) {
	proxier.unmergeService(service, map[ServicePortPortalName]bool{})
}
```

The concept here is the `ServicePortPortalName`, when a new service is added the `proxier.addServicePortPortal` is
called, starting the listener for the new service using the `netsh` tools:

```golang
	args := proxier.netshIPv4AddressAddArgs(serviceIP)
	if existed, err := proxier.netsh.EnsureIPAddress(args, serviceIP); err != nil {
		return nil, err
	} else if !existed {
		klog.V(3).InfoS("Added ip address to fowarder interface for service", "servicePortPortalName", servicePortPortalName.String(), "addr", net.JoinHostPort(listenIP, strconv.Itoa(port)), "protocol", protocol)
	}
```

A new `ProxySocket` listens the proto/port and waits in a `ProxyLoop`, the `tcp.Accept()` blocks until an incoming 
connections comes, the `tryConnect` finds the `loadbalancer.NextEndpoint` available in the round-robin and bumps
the index for the next in the list, this method is still reponsible to handle the affinity logic if this is set.
a `net.DialTimeout` is executed in the endpoint returning the connection object. In the final step, bytes are copied
across both connections.

```golang
go proxyTCP(inConn.(*net.TCPConn), outConn.(*net.TCPConn))
```

`UnmergeServices` does the opposite and calls `closeServicePortPortal`, stopping the connections and cleaning up the interfaces
 with `netsh` commands.

#### Endpoints

There's no `Endpointslices` for this mode, so the logic is kind of simple here
An internal list of endpoints/services are kept on Kube-proxy, they are used when a service connection is made,
and merged on changes in the endpoint slice. A `balancerState` struct is kept internally for each service.

```golang
// LoadBalancerRR is a round-robin load balancer.
type LoadBalancerRR struct {
	lock     sync.RWMutex
	services map[proxy.ServicePortName]*balancerState
}
...
type balancerState struct {
	endpoints []string // a list of "ip:port" style strings
	index     int      // current index into endpoints
	affinity  affinityPolicy
}
```

Adding a new endpoint updates the internal affinity mapping and saves the new endpoints in the correct balancer states,
shuffling the list of endpoints and resetting the index, for update the logic is similar and guarantee the merge.

```golang
	state.endpoints = util.ShuffleStrings(newEndpoints)
	state.index = 0
```

To delete this list of endpoints the service balancer endpoints list is emptied.

## Conclusion 

These two flows allow Kube-Proxy on Windows to work and thus live up to it's proxy terminology.
More advanced, reliable and faster techniques are implemented outside the userland, so when using this
keep in mind the userspace are deprecated and their use still happens in very specific occasions.

## Listening

{{< youtube TwpLzAt4UxM >}}

## References

[1] https://kubernetes.io/docs/concepts/overview/components/#kube-proxy

[2] https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container

[3] https://www.amazon.com/Hands-Kubernetes-Windows-Effectively-orchestrate-ebook/dp/B08576JXTW/

[4] https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-userspace

