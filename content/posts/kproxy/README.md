+++
title = "Kube-proxy Addresses"
description = "ClusterIP and NodePort setup"
date = "2021-06-14"
markup = "mmark"
+++

### Introduction

On this post I will bring a few details around Kube-proxy codebase and, how the rules are setup under the `--mode=iptables`
and a few debugging commands to illustrate all the packet paths and endpoints.

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

$ kubectl get nodes -A -o wide                                                                                                                                                                   [21:42:09]
NAME                    STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
cluster-control-plane   Ready    control-plane,master   26h   v1.21.1   172.18.0.4    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
cluster-worker          Ready    <none>                 26h   v1.21.1   172.18.0.2    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
cluster-worker2         Ready    <none>                 26h   v1.21.1   172.18.0.3    <none>        Ubuntu 21.04   5.8.0-55-generic   containerd://1.5.2
```


-A KUBE-POSTROUTING -m mark ! --mark 0x00004000/0x00004000 -j RETURN
-A KUBE-POSTROUTING -j MARK --xor-mark 0x00004000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
-A KUBE-MARK-MASQ -j MARK --or-mark 0x00004000



Chain PREROUTING (policy ACCEPT 338 packets, 63418 bytes)
pkts bytes target     prot opt in     out     source               destination
383 65148 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

----

-A KUBE-SERVICES -m comment --comment "default/nginx cluster IP" -m tcp -p tcp -d 10.96.34.82/32 --dport 80 -j KUBE-SVC-2CMXP7HKUVJN7L6M

iptables -v --numeric --table nat --list KUBE-SERVICES

Chain KUBE-SERVICES (2 references)
pkts bytes target     prot opt in     out     source               destination
52  3946 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
21  1260 KUBE-SVC-2CMXP7HKUVJN7L6M  tcp  --  *      *       0.0.0.0/0            10.96.34.82          /* default/nginx cluster IP */ tcp dpt:80

-----

--
-A KUBE-SVC-2CMXP7HKUVJN7L6M -m comment --comment "default/nginx cluster IP" -m tcp -p tcp -d 10.96.34.82/32 --dport 80 ! -s 10.244.0.0/16 -j KUBE-MARK-MASQ
-A KUBE-SVC-2CMXP7HKUVJN7L6M -m comment --comment default/nginx -j KUBE-SEP-HUXRPHBUO34UVPEF

iptables -v --numeric --table nat --list KUBE-SVC-2CMXP7HKUVJN7L6M

Chain KUBE-SVC-2CMXP7HKUVJN7L6M (1 references)
pkts bytes target                      prot opt    in     out      source               destination
0     0 KUBE-MARK-MASQ              tcp  --     *      *       !10.244.0.0/16        10.96.34.82          /* default/nginx cluster IP */ tcp dpt:80
21  1260 KUBE-SEP-HUXRPHBUO34UVPEF   all  --     *      *        0.0.0.0/0            0.0.0.0/0            /* default/nginx */
0     0 TRACE                       all  --     *      *        0.0.0.0/0            0.0.0.0/0            /* default/nginx */




---


-A KUBE-SEP-HUXRPHBUO34UVPEF -m comment --comment default/nginx -s 10.244.2.2/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-HUXRPHBUO34UVPEF -m comment --comment default/nginx -m tcp -p tcp -j DNAT --to-destination 10.244.2.2:80

iptables -v --numeric --table nat --list KUBE-SEP-HUXRPHBUO34UVPEF

Chain KUBE-SEP-HUXRPHBUO34UVPEF (1 references)
pkts bytes target     prot opt in     out     source               destination
0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.2.2           0.0.0.0/0            /* default/nginx */
0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx */ tcp to:10.244.2.2:80

