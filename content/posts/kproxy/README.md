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

```
$ kubectl get nodes -A -o wide                                                                                                                                                                   [21:42:09]

NAME                    STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
cluster-control-plane   Ready    control-plane,master   26h   v1.21.1   172.18.0.4    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
cluster-worker          Ready    <none>                 26h   v1.21.1   172.18.0.2    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
cluster-worker2         Ready    <none>                 26h   v1.21.1   172.18.0.3    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
```

Setting up the pods, a client one and a nginx server:

```
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

``` 
$ kubectl get endpoints nginx
NAME    ENDPOINTS       AG
nginx   10.244.2.2:80   2d1h
```

## Connection tests

From the `10.244.1.3` our client gonna hit on nginx on `10.244.2.2`, via the service virtual IP (`10.96.34.82`). 

```
$ host nginx
nginx.default.svc.cluster.local has address 10.96.34.82
$ wget -S -O index.html nginx
Connecting to nginx (10.96.34.82:80)
...
``` 

Running tcpdump in the the server, we can see the correct handshake. 

```
10.244.1.3.37328 > 10.244.2.2.80: Flags [S], cksum 0x191b (incorrect -> 0x1060), seq 1038765636, win 64240, options [mss 1460,sackOK,TS val 3348512221 ecr 0,nop,wscale 7], length 0
10.244.2.2.80 > 10.244.1.3.37328: Flags [S.], cksum 0x191b (incorrect -> 0x8a59), seq 1687129480, ack 1038765637, win 65160, options [mss 1460,sackOK,TS val 3660625428 ecr 3348512221,nop,wscale 7], length 0
10.244.1.3.37328 > 10.244.2.2.80: Flags [.], cksum 0x1913 (incorrect -> 0xb5b8), seq 1, ack 1, win 502, options [nop,nop,TS val 3348512221 ecr 3660625428], length 0
10.244.1.3.37328 > 10.244.2.2.80: Flags [P.], cksum 0x1957 (incorrect -> 0x59ab), seq 1:69, ack 1, win 502, options [nop,nop,TS val 3348512221 ecr 3660625428], length 68: HTTP, length: 68
```

### Kube-Proxy rules

The mode is iptables, accordingly to the documentation: 

```
In this mode, kube-proxy watches the Kubernetes control plane for the addition and removal of Service and Endpoint 
objects. For each Service, it installs iptables rules, which capture traffic to the Service's clusterIP and port, 
and redirect that traffic to one of the Service's backend sets. For each Endpoint object, 
it installs iptables rules which select a backend Pod.
``` 

OK, lets check the rules created by `iptables-save` in the worker, there're some interesting rules created regarding
our service and other paths, a nice diagram around from StackRox [1], shows the proper flow in the chains

{{< img resizedURL="https://res.cloudinary.com/stackrox/v1578619751/kube-networking-5_zswvwp.svg" originalURL="./kproxy.svg" style="width: 100%; " >}}

```
*nat
...
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
...
```

Important things to notice here, the first chain on NAT PREROUTING move packets to the KUBE-SERVICE chain.

```
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
```

This part is important the packet can be marked as MASQ or DROP, a SOURCE NAT Masquerade can happen in
the KUBE-POSTROUTING IF the packet comes from outside the cluster (not from a Pod), meaning the IP source
is going to be the Node ip instead.

Checking the KUBE-SERVICE chain. There's a Target on the `KUBE-SVC-2CMXP7HKUVJN7L6M` chain when the destination is
for IP address of the service (from any source) on TCP 80. 

```
$ iptables -v --numeric --table nat --list KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target                     prot opt in     out     source               destination         
    0     0 KUBE-SVC-2CMXP7HKUVJN7L6M  tcp  --  *      *       0.0.0.0/0            10.96.34.82          /* default/nginx cluster IP */ tcp dpt:80
```

Checking this Chain in particular a few things to consider, 1) the KUBE-MARK-MASQ tag (External SNAT applied) when
the traffic comes from outside the cluster, 2) another JUMP to `KUBE-SEP-HUXRPHBUO34UVPEF`

```
$ iptables -v --numeric --table nat --list KUBE-SVC-2CMXP7HKUVJN7L6M 
Chain KUBE-SVC-2CMXP7HKUVJN7L6M (1 references)
 pkts bytes target                      prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ              tcp  --  *      *      !10.244.0.0/16        10.96.34.82          /* default/nginx cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-HUXRPHBUO34UVPEF   all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx */

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

## References

[1] https://www.stackrox.com/post/2020/01/kubernetes-networking-demystified/
[2] https://courses.academy.tigera.io/courses/course-v1