+++
title = "Network Policy E2E test suite"
description = "Walkthrough on netpol E2E framework"
date = "2020-12-27"
markup = "mmark"
+++

## Network policy

This post documents a few steps to test the E2E framework using the Kind cluster with different
CNIs, giving the developer some good tooling for debugging and quick test replication.

Use this [script](https://github.com/jayunit100/k8sprototypes/blob/master/kind/kind-local-up.sh) to bring a
Kind cluster with a specific CNI setup.

### Finding the config parameters

In the *~/.kube/config* you can find the Cluster sections host, use the IP and Port to start the tests.

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://127.0.0.1:39913
```

### Focused tests

Use this command to start the test suite:

```
KUBERNETES_SERVICE_HOST=127.0.0.1 KUBERNETES_SERVICE_PORT=39913 \
    _output/local/bin/linux/amd64/e2e.test \
    --provider=local \
    --ginkgo.focus="Netpol" \
    --kubeconfig=/root/.kube/config \
    --report-dir=/tmp/output
```

The focus will provide the *Network Policy* under *test/e2e/netpol/network_policy.go* on this namespace:

* **[sig-network] Netpol [LinuxOnly] NetworkPolicy between server and client**

More details and stdout are on *junit_01.xml* files on each folder.

### Antrea

Time: [1451.601s](../output_antrea/junit_01.xml)

Command:
```
sandbox:~$ CLUSTER=antrea ./kind-local-up.sh
```

|     time | failed   | name                                                                                                                                                  |
|---------:|:---------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
|  41.3918 |          | should allow egress access on one named port [Feature:NetworkPolicy]                                                                                  |
|  38.7032 |          | should allow egress access to server in CIDR block [Feature:NetworkPolicy]                                                                            |
|  43.072  |          | should allow ingress access from namespace on one named port [Feature:NetworkPolicy]                                                                  |
|  43.149  |          | should allow ingress access from updated namespace [Feature:NetworkPolicy]                                                                            |
|  41.0726 |          | should allow ingress access from updated pod [Feature:NetworkPolicy]                                                                                  |
|  44.5425 |          | should allow ingress access on one named port [Feature:NetworkPolicy]                                                                                 |
|  43.6281 |          | should deny ingress access to updated pod [Feature:NetworkPolicy]                                                                                     |
|  43.2286 |          | should enforce egress policy allowing traffic to a server in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy] |
|  36.7815 |          | should enforce except clause while egress access to server in CIDR block [Feature:NetworkPolicy]                                                      |
|  42.8188 |          | should enforce multiple egress policies with egress allow-all policy taking precedence [Feature:NetworkPolicy]                                        |
|  54.6405 | *failed* | should enforce multiple ingress policies with ingress allow-all policy taking precedence [Feature:NetworkPolicy]                                      |
|  59.7764 | *failed* | should enforce multiple, stacked policies with overlapping podSelectors [Feature:NetworkPolicy]                                                       |
|  43.7333 |          | should enforce policies to check ingress and egress policies can be controlled independently based on PodSelector [Feature:NetworkPolicy]             |
|  38.2669 |          | should enforce policy based on Multiple PodSelectors and NamespaceSelectors [Feature:NetworkPolicy]                                                   |
|  37.9143 |          | should enforce policy based on NamespaceSelector with MatchExpressions[Feature:NetworkPolicy]                                                         |
|  43.8143 |          | should enforce policy based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]                                                              |
|  37.8272 |          | should enforce policy based on PodSelector or NamespaceSelector [Feature:NetworkPolicy]                                                               |
|  38.5747 |          | should enforce policy based on PodSelector with MatchExpressions[Feature:NetworkPolicy]                                                               |
|  36.7824 |          | should enforce policy based on Ports [Feature:NetworkPolicy]                                                                                          |
|  38.2507 |          | should enforce policy to allow traffic from pods within server namespace based on PodSelector [Feature:NetworkPolicy]                                 |
|  38.7376 |          | should enforce policy to allow traffic only from a different namespace, based on NamespaceSelector [Feature:NetworkPolicy]                            |
|  37.2833 |          | should enforce policy to allow traffic only from a pod in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]    |
|  49.4517 | *failed* | should enforce updated policy [Feature:NetworkPolicy]                                                                                                 |
|  43.1706 |          | should ensure an IP overlapping both IPBlock.CIDR and IPBlock.Except is allowed [Feature:NetworkPolicy]                                               |
|  37.1481 |          | should not allow access by TCP when a policy specifies only SCTP [Feature:NetworkPolicy] [Feature:SCTP]                                               |
|  38.1034 |          | should not allow access by TCP when a policy specifies only UDP [Feature:NetworkPolicy] [Feature:UDP]                                                 |
| 189.733  |          | should stop enforcing policies after they are deleted [Feature:NetworkPolicy]                                                                         |
|  39.0466 |          | should support a &#x27;default-deny-all&#x27; policy [Feature:NetworkPolicy]                                                                          |
|  44.9421 | *failed* | should support a &#x27;default-deny-ingress&#x27; policy [Feature:NetworkPolicy]                                                                      |
|  41.5988 |          | should support allow-all policy [Feature:NetworkPolicy]                                                                                               |
|  44.2273 |          | should work with Ingress, Egress specified together [Feature:NetworkPolicy]                                                                           |

### Cilium

Time: [1370.553s](../output_cilium/junit_01.xml)

Command:
```
sandbox:~$ CLUSTER=cilium ./kind-local-up.sh
```

|     time | failed   | name                                                                                                                                                  |
|---------:|:---------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
|  43.7662 |          | should allow egress access on one named port [Feature:NetworkPolicy]                                                                                  |
|  43.4651 | *failed* | should allow egress access to server in CIDR block [Feature:NetworkPolicy]                                                                            |
|  43.7539 |          | should allow ingress access from namespace on one named port [Feature:NetworkPolicy]                                                                  |
|  50.204  | *failed* | should allow ingress access from updated namespace [Feature:NetworkPolicy]                                                                            |
|  48.8958 | *failed* | should allow ingress access from updated pod [Feature:NetworkPolicy]                                                                                  |
|  43.9659 |          | should allow ingress access on one named port [Feature:NetworkPolicy]                                                                                 |
|  46.3589 | *failed* | should deny ingress access to updated pod [Feature:NetworkPolicy]                                                                                     |
|  36.8897 |          | should enforce egress policy allowing traffic to a server in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy] |
|  44.3379 | *failed* | should enforce except clause while egress access to server in CIDR block [Feature:NetworkPolicy]                                                      |
|  40.0372 |          | should enforce multiple egress policies with egress allow-all policy taking precedence [Feature:NetworkPolicy]                                        |
|  43.7602 |          | should enforce multiple ingress policies with ingress allow-all policy taking precedence [Feature:NetworkPolicy]                                      |
|  52.7325 |          | should enforce multiple, stacked policies with overlapping podSelectors [Feature:NetworkPolicy]                                                       |
|  42.2191 |          | should enforce policies to check ingress and egress policies can be controlled independently based on PodSelector [Feature:NetworkPolicy]             |
|  38.8015 |          | should enforce policy based on Multiple PodSelectors and NamespaceSelectors [Feature:NetworkPolicy]                                                   |
|  36.4353 |          | should enforce policy based on NamespaceSelector with MatchExpressions[Feature:NetworkPolicy]                                                         |
|  39.2781 |          | should enforce policy based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]                                                              |
|  38.684  |          | should enforce policy based on PodSelector or NamespaceSelector [Feature:NetworkPolicy]                                                               |
|  39.2452 |          | should enforce policy based on PodSelector with MatchExpressions[Feature:NetworkPolicy]                                                               |
|  39.326  |          | should enforce policy based on Ports [Feature:NetworkPolicy]                                                                                          |
|  38.603  |          | should enforce policy to allow traffic from pods within server namespace based on PodSelector [Feature:NetworkPolicy]                                 |
|  39.2863 |          | should enforce policy to allow traffic only from a different namespace, based on NamespaceSelector [Feature:NetworkPolicy]                            |
|  39.1103 |          | should enforce policy to allow traffic only from a pod in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]    |
|  40.9576 |          | should enforce updated policy [Feature:NetworkPolicy]                                                                                                 |
|  41.2865 | *failed* | should ensure an IP overlapping both IPBlock.CIDR and IPBlock.Except is allowed [Feature:NetworkPolicy]                                               |
|  42.2095 | *failed* | should not allow access by TCP when a policy specifies only SCTP [Feature:NetworkPolicy] [Feature:SCTP]                                               |
|  37.474  |          | should not allow access by TCP when a policy specifies only UDP [Feature:NetworkPolicy] [Feature:UDP]                                                 |
| 112.803  |          | should stop enforcing policies after they are deleted [Feature:NetworkPolicy]                                                                         |
|  41.3663 |          | should support a &#x27;default-deny-all&#x27; policy [Feature:NetworkPolicy]                                                                          |
|  39.0023 |          | should support a &#x27;default-deny-ingress&#x27; policy [Feature:NetworkPolicy]                                                                      |
|  42.1974 |          | should support allow-all policy [Feature:NetworkPolicy]                                                                                               |
|  43.8719 |          | should work with Ingress, Egress specified together [Feature:NetworkPolicy]                                                                           |

### Calico

Time: [1324.684s](../output_calico/junit_01.xml)

Command:
```
sandbox:~$ CLUSTER=calico ./kind-local-up.sh
```

|     time | failed   | name                                                                                                                                                  |
|---------:|:---------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
|  35.8095 |          | should allow egress access on one named port [Feature:NetworkPolicy]                                                                                  |
|  34.0198 |          | should allow egress access to server in CIDR block [Feature:NetworkPolicy]                                                                            |
|  38.3741 |          | should allow ingress access from namespace on one named port [Feature:NetworkPolicy]                                                                  |
|  42.5764 |          | should allow ingress access from updated namespace [Feature:NetworkPolicy]                                                                            |
|  72.1853 |          | should allow ingress access from updated pod [Feature:NetworkPolicy]                                                                                  |
|  44.9761 |          | should allow ingress access on one named port [Feature:NetworkPolicy]                                                                                 |
|  37.9649 |          | should deny ingress access to updated pod [Feature:NetworkPolicy]                                                                                     |
|  37.2776 |          | should enforce egress policy allowing traffic to a server in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy] |
|  34.4918 |          | should enforce except clause while egress access to server in CIDR block [Feature:NetworkPolicy]                                                      |
|  42.5749 |          | should enforce multiple egress policies with egress allow-all policy taking precedence [Feature:NetworkPolicy]                                        |
|  39.7623 |          | should enforce multiple ingress policies with ingress allow-all policy taking precedence [Feature:NetworkPolicy]                                      |
|  45.2768 |          | should enforce multiple, stacked policies with overlapping podSelectors [Feature:NetworkPolicy]                                                       |
|  37.146  |          | should enforce policies to check ingress and egress policies can be controlled independently based on PodSelector [Feature:NetworkPolicy]             |
|  39.3511 |          | should enforce policy based on Multiple PodSelectors and NamespaceSelectors [Feature:NetworkPolicy]                                                   |
|  34.7001 |          | should enforce policy based on NamespaceSelector with MatchExpressions[Feature:NetworkPolicy]                                                         |
|  35.1533 |          | should enforce policy based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]                                                              |
|  35.1159 |          | should enforce policy based on PodSelector or NamespaceSelector [Feature:NetworkPolicy]                                                               |
|  41.341  |          | should enforce policy based on PodSelector with MatchExpressions[Feature:NetworkPolicy]                                                               |
|  38.4367 |          | should enforce policy based on Ports [Feature:NetworkPolicy]                                                                                          |
|  56.606  |          | should enforce policy to allow traffic from pods within server namespace based on PodSelector [Feature:NetworkPolicy]                                 |
|  35.0263 |          | should enforce policy to allow traffic only from a different namespace, based on NamespaceSelector [Feature:NetworkPolicy]                            |
|  39.1988 |          | should enforce policy to allow traffic only from a pod in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]    |
|  38.3293 |          | should enforce updated policy [Feature:NetworkPolicy]                                                                                                 |
|  38.7177 |          | should ensure an IP overlapping both IPBlock.CIDR and IPBlock.Except is allowed [Feature:NetworkPolicy]                                               |
|  42.6838 | *failed* | should not allow access by TCP when a policy specifies only SCTP [Feature:NetworkPolicy] [Feature:SCTP]                                               |
|  33.4298 |          | should not allow access by TCP when a policy specifies only UDP [Feature:NetworkPolicy] [Feature:UDP]                                                 |
| 108.302  |          | should stop enforcing policies after they are deleted [Feature:NetworkPolicy]                                                                         |
|  41.1466 |          | should support a &#x27;default-deny-all&#x27; policy [Feature:NetworkPolicy]                                                                          |
|  38.3876 |          | should support a &#x27;default-deny-ingress&#x27; policy [Feature:NetworkPolicy]                                                                      |
|  41.9732 |          | should support allow-all policy [Feature:NetworkPolicy]                                                                                               |
|  44.1808 |          | should work with Ingress, Egress specified together [Feature:NetworkPolicy]                                                                           |

### No CNI

Time: [1093.335](../output_nocni/junit_01.xml)

Command (without Netpol support):
```
sandbox:~$ kind cluster create
```

|    time | failed   | name                                                                                                                                                  |
|--------:|:---------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
| 36.8562 | *failed* | should allow egress access on one named port [Feature:NetworkPolicy]                                                                                  |
| 37.117  | *failed* | should allow egress access to server in CIDR block [Feature:NetworkPolicy]                                                                            |
| 33.6975 | *failed* | should allow ingress access from namespace on one named port [Feature:NetworkPolicy]                                                                  |
| 37.4686 | *failed* | should allow ingress access from updated namespace [Feature:NetworkPolicy]                                                                            |
| 32.0948 | *failed* | should allow ingress access from updated pod [Feature:NetworkPolicy]                                                                                  |
| 36.5492 | *failed* | should allow ingress access on one named port [Feature:NetworkPolicy]                                                                                 |
| 36.3705 | *failed* | should deny ingress access to updated pod [Feature:NetworkPolicy]                                                                                     |
| 37.9346 | *failed* | should enforce egress policy allowing traffic to a server in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy] |
| 33.477  | *failed* | should enforce except clause while egress access to server in CIDR block [Feature:NetworkPolicy]                                                      |
| 33.6098 | *failed* | should enforce multiple egress policies with egress allow-all policy taking precedence [Feature:NetworkPolicy]                                        |
| 38.2607 | *failed* | should enforce multiple ingress policies with ingress allow-all policy taking precedence [Feature:NetworkPolicy]                                      |
| 33.4111 | *failed* | should enforce multiple, stacked policies with overlapping podSelectors [Feature:NetworkPolicy]                                                       |
| 36.2295 | *failed* | should enforce policies to check ingress and egress policies can be controlled independently based on PodSelector [Feature:NetworkPolicy]             |
| 31.8398 | *failed* | should enforce policy based on Multiple PodSelectors and NamespaceSelectors [Feature:NetworkPolicy]                                                   |
| 33.3985 | *failed* | should enforce policy based on NamespaceSelector with MatchExpressions[Feature:NetworkPolicy]                                                         |
| 33.3989 | *failed* | should enforce policy based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]                                                              |
| 32.5186 | *failed* | should enforce policy based on PodSelector or NamespaceSelector [Feature:NetworkPolicy]                                                               |
| 33.3631 | *failed* | should enforce policy based on PodSelector with MatchExpressions[Feature:NetworkPolicy]                                                               |
| 32.8972 | *failed* | should enforce policy based on Ports [Feature:NetworkPolicy]                                                                                          |
| 32.2191 | *failed* | should enforce policy to allow traffic from pods within server namespace based on PodSelector [Feature:NetworkPolicy]                                 |
| 38.2643 | *failed* | should enforce policy to allow traffic only from a different namespace, based on NamespaceSelector [Feature:NetworkPolicy]                            |
| 33.6517 | *failed* | should enforce policy to allow traffic only from a pod in a different namespace based on PodSelector and NamespaceSelector [Feature:NetworkPolicy]    |
| 40.643  | *failed* | should enforce updated policy [Feature:NetworkPolicy]                                                                                                 |
| 32.9202 | *failed* | should ensure an IP overlapping both IPBlock.CIDR and IPBlock.Except is allowed [Feature:NetworkPolicy]                                               |
| 42.4549 | *failed* | should not allow access by TCP when a policy specifies only SCTP [Feature:NetworkPolicy] [Feature:SCTP]                                               |
| 37.3655 | *failed* | should not allow access by TCP when a policy specifies only UDP [Feature:NetworkPolicy] [Feature:UDP]                                                 |
| 38.5486 | *failed* | should stop enforcing policies after they are deleted [Feature:NetworkPolicy]                                                                         |
| 33.6928 | *failed* | should support a &#x27;default-deny-all&#x27; policy [Feature:NetworkPolicy]                                                                          |
| 38.0615 | *failed* | should support a &#x27;default-deny-ingress&#x27; policy [Feature:NetworkPolicy]                                                                      |
| 31.8662 |          | should support allow-all policy [Feature:NetworkPolicy]                                                                                               |
| 32.9995 | *failed* | should work with Ingress, Egress specified together [Feature:NetworkPolicy]  

## More information

* @jayunit100 - test matrix - https://github.com/kubernetes/kubernetes/pull/91592
* Initial setup post - https://jayunit100.blogspot.com/2020/05/visualize-networkpolicies-as-truthtables.html