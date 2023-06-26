+++
title = "Kube-proxy ClusterIP"
description = "ClusterIP iptables configurations"
date = "2021-06-14"
markup = "mmark"
+++

## Introduction

On this post I will bring a few details around Kube-proxy codebase and, how the rules are setup under the `--mode=iptables`
and a few debugging commands to illustrate all the packet paths and endpoints. There's a simple cluster with 3 nodes 
on v1.21.1 created with Kind and kindnet as CNI.

## Setup

These are the following ips:

```shell
$ kubectl get nodes -A -o wide                                                                                                                                                                   [21:42:09]

NAME                    STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
cluster-control-plane   Ready    control-plane,master   26h   v1.21.1   172.18.0.4    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
cluster-worker          Ready    <none>                 26h   v1.21.1   172.18.0.2    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
cluster-worker2         Ready    <none>                 26h   v1.21.1   172.18.0.3    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
```

Setting up the pods, a client one and a nginx server:

```shell
$ kubectl run client --image=busybox --restart=Never -- sleep 1d
$ kubectl run nginx --image=nginx
$ kubectl expose pod/nginx --name nginx --port 80

$ kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE     IP           NODE              NOMINATED NODE   READINESS GATES
client   1/1     Running   0          7m44s   10.244.1.3   cluster-worker    <none>           <none>
nginx    1/1     Running   0          25h     10.244.2.2   cluster-worker2   <none>           <none>

$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx        ClusterIP   10.96.34.82     <none>        80/TCP    26h
```

So we have Pod client on `10.244.1.3`, and nginx on `10.244.2.2` and the service running on ClusterIP `10.96.34.82`
this is a virtual IP address, and as we can see will be used to load balance the traffic between the pods
behind the service.

Lets take a look in the Endpoint object, as we can see the service is: 

```shell
$ kubectl get endpoints nginx
NAME    ENDPOINTS       AG
nginx   10.244.2.2:80   2d1h
```

## Connection tests

From the `10.244.1.3` our client gonna hit on nginx on `10.244.2.2`, via the service virtual IP (`10.96.34.82`). 

```shell
$ host nginx
nginx.default.svc.cluster.local has address 10.96.34.82
$ wget -S -O index.html nginx
Connecting to nginx (10.96.34.82:80)
...
``` 

Running tcpdump in the the server, we can see the correct handshake. 

```shell
10.244.1.3.37328 > 10.244.2.2.80: Flags [S], cksum 0x191b (incorrect -> 0x1060), seq 1038765636, win 64240, options [mss 1460,sackOK,TS val 3348512221 ecr 0,nop,wscale 7], length 0
10.244.2.2.80 > 10.244.1.3.37328: Flags [S.], cksum 0x191b (incorrect -> 0x8a59), seq 1687129480, ack 1038765637, win 65160, options [mss 1460,sackOK,TS val 3660625428 ecr 3348512221,nop,wscale 7], length 0
10.244.1.3.37328 > 10.244.2.2.80: Flags [.], cksum 0x1913 (incorrect -> 0xb5b8), seq 1, ack 1, win 502, options [nop,nop,TS val 3348512221 ecr 3660625428], length 0
10.244.1.3.37328 > 10.244.2.2.80: Flags [P.], cksum 0x1957 (incorrect -> 0x59ab), seq 1:69, ack 1, win 502, options [nop,nop,TS val 3348512221 ecr 3660625428], length 68: HTTP, length: 68
```

## Kube-Proxy rules and chains

The mode is iptables, accordingly to the documentation: 

```
In this mode, kube-proxy watches the Kubernetes control plane for the addition and removal of Service and Endpoint 
objects. For each Service, it installs iptables rules, which capture traffic to the Service's clusterIP and port, 
and redirect that traffic to one of the Service's backend sets. For each Endpoint object, 
it installs iptables rules which select a backend Pod.
``` 

OK, lets check the rules created by `iptables-save` in the worker, there're some interesting rules created regarding
our service and other paths, a nice diagram around from StackRox [1], shows the proper flow in the chains

![screen1](https://res.cloudinary.com/stackrox/v1578619751/kube-networking-5_zswvwp.svg?width=1024 "screen1")

```shell
*nat
...
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
...
```

Important things to notice here, the first chain on NAT PREROUTING move packets to the KUBE-SERVICE chain.

```shell
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
```

These rules are grant on `syncProxyRules:871`, before each synchronization:

https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go#L953

{{<highlight golang>}}
var iptablesJumpChains = []iptablesJumpChain{
	{utiliptables.TableFilter, kubeExternalServicesChain, utiliptables.ChainInput, "kubernetes externally-visible service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeExternalServicesChain, utiliptables.ChainForward, "kubernetes externally-visible service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeNodePortsChain, utiliptables.ChainInput, "kubernetes health check service ports", nil},
	{utiliptables.TableFilter, kubeServicesChain, utiliptables.ChainForward, "kubernetes service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeServicesChain, utiliptables.ChainOutput, "kubernetes service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeForwardChain, utiliptables.ChainForward, "kubernetes forwarding rules", nil},
	{utiliptables.TableNAT, kubeServicesChain, utiliptables.ChainOutput, "kubernetes service portals", nil},
	{utiliptables.TableNAT, kubeServicesChain, utiliptables.ChainPrerouting, "kubernetes service portals", nil},
	{utiliptables.TableNAT, kubePostroutingChain, utiliptables.ChainPostrouting, "kubernetes postrouting rules", nil},
}

var iptablesEnsureChains = []struct {
	table utiliptables.Table
	chain utiliptables.Chain
}{
	{utiliptables.TableNAT, KubeMarkDropChain},
}

func (proxier *Proxier) syncProxyRules() {
...
	// Create and link the kube chains.
	for _, jump := range iptablesJumpChains {
		if _, err := proxier.iptables.EnsureChain(jump.table, jump.dstChain); err != nil {
			klog.ErrorS(err, "Failed to ensure chain exists", "table", jump.table, "chain", jump.dstChain)
			return
		}
		args := append(jump.extraArgs,
			"-m", "comment", "--comment", jump.comment,
			"-j", string(jump.dstChain),
		)
		if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, jump.table, jump.srcChain, args...); err != nil {
			klog.ErrorS(err, "Failed to ensure chain jumps", "table", jump.table, "srcChain", jump.srcChain, "dstChain", jump.dstChain)
			return
		}
	}

	// ensure KUBE-MARK-DROP chain exist but do not change any rules
	for _, ch := range iptablesEnsureChains {
		if _, err := proxier.iptables.EnsureChain(ch.table, ch.chain); err != nil {
			klog.ErrorS(err, "Failed to ensure chain exists", "table", ch.table, "chain", ch.chain)
			return
		}
	}
...
	masqRule := []string{
		"-A", string(kubePostroutingChain),
		"-m", "comment", "--comment", `"kubernetes service traffic requiring SNAT"`,
		"-j", "MASQUERADE",
	}
	if proxier.iptables.HasRandomFully() {
		masqRule = append(masqRule, "--random-fully")
	}
	utilproxy.WriteLine(proxier.natRules, masqRule...)
...

{{</highlight>}}



This part is important the packet can be marked as `KUBE-MARK-MASQ` or `KUBE-MARK-DROP`, a SOURCE NAT Masquerade can happen in
the `KUBE-POSTROUTING` IF the packet comes from outside the cluster (not from a Pod), meaning the IP source
is going to be the Node ip instead.

Checking the KUBE-SERVICE chain. There's a Target on the `KUBE-SVC-2CMXP7HKUVJN7L6M` chain when the destination is
for IP address of the service (from any source) on TCP 80. 

```shell
$ iptables -v --numeric --table nat --list KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target                     prot opt in     out     source               destination         
    0     0 KUBE-SVC-2CMXP7HKUVJN7L6M  tcp  --  *      *       0.0.0.0/0            10.96.34.82          /* default/nginx cluster IP */ tcp dpt:80
```

These services rule for ClusterIP starts to be created in this mais loop

https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go#L1015

{{<highlight golang>}}
	for svcName, svc := range proxier.serviceMap {
		hasEndpoints := len(readyEndpoints) > 0	
		if hasEndpoints {  // create the service rules, jumping to SVC chain
			// proxier.masqueradeAll

			// -A KUBE-SVC-2CMXP7HKUVJN7L6M ! -s 10.244.0.0/16 -d 10.96.34.82/32 -p tcp -m comment --comment "default/nginx cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
			// -A KUBE-SERVICES -d 10.96.34.82/32 -p tcp -m comment --comment "default/nginx cluster IP" -m tcp --dport 80 -j KUBE-SVC-2CMXP7HKUVJN7L6M
		} else {
		    -j REJECT
		}
	}
{{</highlight>}}


Checking this Chain in particular a few things to consider, 1) the KUBE-MARK-MASQ tag (External SNAT applied) when
the traffic comes from outside the cluster, 2) another JUMP to `KUBE-SEP-HUXRPHBUO34UVPEF`

```
$ iptables -v --numeric --table nat --list KUBE-SVC-2CMXP7HKUVJN7L6M 
Chain KUBE-SVC-2CMXP7HKUVJN7L6M (1 references)
 pkts bytes target                      prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ              tcp  --  *      *      !10.244.0.0/16        10.96.34.82          /* default/nginx cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-HUXRPHBUO34UVPEF   all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx */

```

This iteration inside the service loop, brings the creation of new chains for final DNAT and the jump between them
 as seen in the last rule:

https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go#L1407

{{<highlight golang>}}
// filling the endpoint chains from ready endpoints for this service.
for _, ep := range readyEndpoints {
	endpointChain = epInfo.endpointChain(svcNameString, protocol)
	endpointChains = append(endpointChains, endpointChain)
}

for i, endpointChain := range endpointChains {
	args = proxier.appendServiceCommentLocked(args, svcNameString)
	if i < (n - 1) {
		// Each rule is a probabilistic match.
		args = append(args,
			"-m", "statistic",
			"--mode", "random",
			"--probability", proxier.probability(n-i))
	}
	// The final (or only if n == 1) rule is a guaranteed match.
	args = append(args, "-j", string(endpointChain))
	utilproxy.WriteLine(proxier.natRules, args...)
{{</highlight>}}
```
// -A KUBE-SVC-2CMXP7HKUVJN7L6M -m comment --comment default/nginx -j KUBE-SEP-HUXRPHBUO34UVPEF
```

Finally, the KUBE-SEP chain finalized with 1) Ensure the SNAT happens, 2) Finally a DNAT is created, translating
the CluserIP address to the final POD IP address, iptables will ensure a load balancing with probabilities
if multiple pod exists under this service.

```
$ iptables -v --numeric --table nat --list KUBE-SEP-HUXRPHBUO34UVPEF
Chain KUBE-SEP-HUXRPHBUO34UVPEF (1 references)
pkts bytes target       prot opt in     out     source               destination
0     0    KUBE-MARK-MASQ  all  --  *      *       10.244.2.2           0.0.0.0/0            /* default/nginx */
0     0    DNAT            tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx */ tcp to:10.244.2.2:80
```

Yet, for each service the DNAT is made with the proper probability and service port endpoint translation:

https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go#L1444

{{<highlight golang>}}
// Handle traffic that loops back to the originator with SNAT.
utilproxy.WriteLine(proxier.natRules, append(args,
	"-s", utilproxy.ToCIDR(net.ParseIP(epIP)),
	"-j", string(KubeMarkMasqChain))...)
{{</highlight>}}

```
// -A KUBE-SEP-HUXRPHBUO34UVPEF -m comment --comment default/nginx -s 10.244.2.2/32 -j KUBE-MARK-MASQ
```

{{<highlight golang>}}
// DNAT to final destination.
args = append(args, "-m", protocol, "-p", protocol, "-j", "DNAT", "--to-destination", endpoints[i].Endpoint)
utilproxy.WriteLine(proxier.natRules, args...)
{{</highlight>}}

```
// -A KUBE-SEP-HUXRPHBUO34UVPEF -m comment --comment default/nginx -m tcp -p tcp -j DNAT --to-destination 10.244.2.2:80
```

## References

[1] https://www.stackrox.com/post/2020/01/kubernetes-networking-demystified/

[2] https://courses.academy.tigera.io/courses/course-v1
