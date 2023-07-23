+++
title = "Windows and Calico overlay networking"
description = "Notes on Windows and Calico CNI overlay connectivity"
date = "2021-09-03"
markup = "mmark"
+++

## Introduction

Windows are nowadays a must in a lot of multi-os clusters around companies, all big providers
support the creation of Windows server machines and the support of containers in this OS is
getting stronger on a daily basis [1].

The Kubernetes community have the Windows Special Interest Group [2], focusing on workloads
and scheduling support for these Windows nodes. As support tooling and sub-projects around the main
SIGs on K8s, a few experimentations are born like the Dev-Tools [3].

At this point the focus of this post will be debugging an issue found in the tooling environment
involving Calico and Vagrant setup. The content here exposes the steps followed to find
the problem and the aggregation of posts and material used in the research of solving this particular
problem. Keep reading, because interesting procedures and notes are about to come.

## Windows vs linux network namespace

A brief overview and mappings from the similarities between both operating systems when talking
about containers workloads, as expected a few fundamentals differences exists, and the table below
maps what they are:

<center>

|         Linux         |     Windows                 |
|-----------------------|-----------------------------|
| Network namespace     | TCP/IP Compartments         |
| Bridge / IP Routing   | vSwitch                     |
| IP links              | vNICs / vSwitch / VFP ports |
| IPVS and iptables     | Firewall and VFP rules      |

</center>

On my [last post](https://opssec/2021/0828/) I explored some network namespace tooling
in the Linux box, if you have interest to test it.

The debugging on this post goes to VFP and vSwitch, you can read more about it [here](https://www.microsoft.com/en-us/research/publication/vfp-virtual-switch-platform-host-sdn-public-cloud-article-version/).

### Vagrant

It is used in the dev environment and here comes the tricky part, by default it creates an internal
interface with the IP address `10.0.2.15`, and gateway IP address `10.0.2.2`. This is true for both boxes (controlplane
and windows host), so the chance is that most services are default listening to this interface.

On the other side to avoid this internal networking usage, the development environment brings up another 
private network, it is using the IP address range on `10.20.30.0/24`, with last octect `.10` for the control plane
and `.11` for the windows node. 

To summarize this post explores connectivity problems between both hosts related with them having an internal,
and a public NIC at the same time.

### Overlay networking

A note about overlay networking and more details of this setup. the Calico CNI used in the environment is running with 
VXLAN overlay routing mode. So all nodes are communicating via tunneling on port UDP 4789.

A few words from the Project Calico VXLAN [4]:

```
An overlay network allows network devices to communicate across an underlying network (referred to as the underlay)
without the underlay network having any knowledge of the devices connected to the overlay network.

There are many different kinds of overlay networks that use different protocols to make this happen, 
but in general they share the same common characteristic of taking a network packet, referred to as the inner packet,
and encapsulating it inside an outer network packet. 

In this way the underlay sees the outer packets without needing to understand how to handle the inner packets.
```

Felix must be configured for this routing mode, you can find more information of this in the official CNI 
documentation linked here.

## Testing connectivity

### Control plane node 

With all machines up the first basic connectivity test (smoke test) was starting a new Netshoot pod
in the Linux machine and `curl` the Windows IIS IP, which is in `Running` state.

```shell
vagrant@controlplane:~$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS      AGE     IP               NODE           NOMINATED NODE   READINESS GATES
netshoot                              1/1     Running   0             2m28s   100.244.49.69    controlplane   <none>           <none>
nginx-deployment-76d8669c4c-mv4wh     1/1     Running   0             117m    100.244.49.68    controlplane   <none>           <none>
windows-server-iis-769d57f976-nx7hf   1/1     Running   2 (39s ago)   3m3s    100.244.206.68   winw1          <none>           <none>


vagrant@controlplane:~$ kubectl exec -it netshoot -- bash
bash-5.1# curl 100.244.206.68
```

No luck here, it ended up in a timeout after some time, still in the Linux node running the `tshark`, it was
possible to inspect the packet, without a SYN/ACK from the Windows node:

```shell
# tshark -i any
  ...
  1 0.000000000 100.244.49.69 → 100.244.206.68 TCP 74 54398 → 80 [SYN] Seq=0 Win=64860 Len=0 MSS=1410 SACK_PERM=1 TSval=2598139194 TSecr=0 WS=128
  2 1.008358160 100.244.49.69 → 100.244.206.68 TCP 74 [TCP Retransmission] 54398 → 80 [SYN] Seq=0 Win=64860 Len=0 MSS=1410 SACK_PERM=1 TSval=2598140202 TSecr=0 WS=128
  ...
```

Still in the Linux machine, the last try was to check the routes, as we can see in the last line 
the next hop to handle it is `100.244.206.65` via `vxlan.calico`, or the overlay interface.

```shell
vagrant@controlplane:~$ ip route

default via 10.0.2.2 dev eth0 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15
10.0.2.2 dev eth0 proto dhcp scope link src 10.0.2.15 metric 100
10.20.30.0/24 dev eth1 proto kernel scope link src 10.20.30.10
100.244.49.65 dev cali64856e06016 scope link
100.244.49.66 dev cali4bfaac1b56f scope link
100.244.49.67 dev cali897f6784663 scope link
100.244.49.68 dev calide492000d53 scope link
100.244.49.69 dev calie348bef9edc scope link
100.244.206.64/26 via 100.244.206.65 dev vxlan.calico onlink
```

The question now: where is this next hop? In the last post we saw the CNI created these routing rules and normally the
IP CIDR destination was pointing to the node gateway owner of this CIDR, so lets take a look in the windows machine.

### Windows node

Starting with Calico Node logs under `c:\CalicoWindows\logs` to see if it was possible to fetch more
information about the vxlan interface and this strange IP created, found out this piece of log:

```shell
Management IP detected on vSwitch: 10.0.2.15.
```

That's odd, we are not supposed to use this IP anywhere in the setup, and instead the NIC should be the
private one under `10.20.30.11` ip address. This is where Kubelet is listening on, so Calico should be too.
The first procedure here was to take a look in the created Overlay HNS networks with the Powershell
function `Get-HNSNetwork`:

```shell
PS C:\> Get-HNSNetwork

ActivityId             : 20CE6C3F-97EA-4349-A4B2-3861A1B48B5E
AdditionalParams       :
CurrentEndpointCount   : 1
DNSServerCompartment   : 4
DrMacAddress           : 00-15-5D-64-51-0E
Extensions             : {@{Id=E7C3B2F0-F3C5-48DF-AF2B-10FED6D72E7A; IsEnabled=False; Name=Microsoft Windows Filtering Platform}, @{Id=E9B59CFA-2BE1-4B21-828F-B6FBDBDDC017; IsEnabled=True;
                         Name=Microsoft Azure VFP Switch Extension}, @{Id=EA24CD6C-D17A-4348-9190-09F0D5BE83DD; IsEnabled=True; Name=Microsoft NDIS Capture}}
Flags                  : 0
Health                 : @{LastErrorCode=0; LastUpdateTime=132749223763450147}
ID                     : 28893E01-AC8D-4BE6-85D4-7CD3A78611C1
IPv6                   : False
LayeredOn              : B84E00EB-D250-4433-A3DE-E2FC84C23386
MacPools               : {@{EndMacAddress=00-15-5D-D3-3F-FF; StartMacAddress=00-15-5D-D3-30-00}}
ManagementIP           : 10.0.2.15
MaxConcurrentEndpoints : 1
Name                   : Calico
Policies               : {@{Type=HostRoute}, @{DestinationPrefix=100.244.49.64/26; DistributedRouterMacAddress=66-a5-cc-c1-86-97; IsolationId=4096; ProviderAddress=10.20.30.10; Type=RemoteSubnetRoute}}
Resources              : @{AdditionalParams=; AllocationOrder=1; Allocators=System.Object[]; Health=; ID=20CE6C3F-97EA-4349-A4B2-3861A1B48B5E; PortOperationTime=0; State=1; SwitchOperationTime=0;
                         VfpOperationTime=0; parentId=FE09F4C4-1515-4AA7-AB82-A817B557DFBC}
State                  : 1
Subnets                : {@{AdditionalParams=; AddressPrefix=100.244.206.64/26; GatewayAddress=100.244.206.65; Health=; ID=5E1D8701-E648-441B-87A7-3AA5347147E3; ObjectType=5; Policies=System.Object[];
                         State=0}}
TotalEndpoints         : 1
Type                   : Overlay
Version                : 38654705669
```

Ok, so at this point we found out the subnet on Subnets fields, and it was clear the wrong interface
was being used. The reason is shown in the Policies `Host-Route` item for the Linux machine,
Linux is using the `10.20.30.10` interface to setup the VXLAN tunnel on it's side. 

To ensure the proper routing between both hosts the last question that remaining is:

*How to choose the VXLAN interface for this Windows node?*

### Connectivity tests and debugging

After looking around the [New-HnsNetwork](https://github.com/microsoft/SDN/blob/master/Kubernetes/windows/hns.psm1#L148)
a parameter called `AdapterName` exists. It was learned after further analysis that the function is used
by Calico to [bring the Overlay interface](https://github.com/projectcalico/node/blob/master/windows-packaging/CalicoWindows/node/node-service.ps1#L91).
It was a matter of passing the `-AdapterName` as a parameter, like:

```
if ($env:CALICO_NETWORKING_BACKEND -EQ "vxlan") {
    New-NetFirewallRule -Name OverlayTraffic4789UDP -Description "Overlay network traffic UDP" ... // firewall rule
    $result = New-HNSNetwork -Type Overlay ... -AdapterName $vxlanAdapter -Verbose
}
```

Now it's possible to configure this adapter on `config.ps1`, this option should be available on `Calico v3.21.0`:

```
VXLAN_ADAPTER = "Ethernet 2"
```

Let's take a look into the Calico logs again, the private interface is now the Management IP on the vSwitch,
this is a good signal:

```shell
...
Management IP detected on vSwitch: 10.20.30.11
...
2021-08-31 17:54:59.464 [INFO][5524] startup/startup_windows.go 66: Backend networking is vxlan, ensure vxlan network.
2021-08-31 17:54:59.465 [INFO][5524] startup/ipam.go 1895: Ensure block for host winw1, ipv4 attr &{3 1 windows-reserved-ipam-handle windows host rsvd} ipv6 attr <nil>
2021-08-31 17:54:59.465 [INFO][5524] startup/ipam.go 1966: Looking up existing affinities for host host="winw1"
2021-08-31 17:54:59.470 [INFO][5524] startup/ipam.go 356: Looking up existing affinities for host host="winw1"
2021-08-31 17:54:59.473 [INFO][5524] startup/ipam.go 439: Trying affinity for 100.244.206.64/26 host="winw1"
2021-08-31 17:54:59.475 [INFO][5524] startup/ipam.go 143: Attempting to load block cidr=100.244.206.64/26 host="winw1"
2021-08-31 17:54:59.479 [INFO][5524] startup/ipam.go 220: Affinity is confirmed and block has been loaded cidr=100.244.206.64/26 host="winw1"
2021-08-31 17:54:59.479 [INFO][5524] startup/ipam.go 1992: Host's block '100.244.206.64/26'  host="winw1"
2021-08-31 17:54:59.491 [INFO][5524] startup/dataplane_windows.go 357: Attempting to create HNSNetwork {"Name":"Calico","Type":"Overlay","Subnets":[{"AddressPrefix":"100.244.206.64/26","GatewayAddress":"1
00.244.206.65","Policies":[{"Type":"VSID","VSID":4096}]}]} subnet="100.244.206.64/26"
```

Checking the Overlay Network again, looks promising:

```shell
PS C:\> Get-HNSNetwork

ActivityId             : DE451C81-5986-4BF9-B720-2611F5DEE310
AdditionalParams       :
CurrentEndpointCount   : 1
DNSServerCompartment   : 4
DrMacAddress           : 00-15-5D-B9-70-84
Extensions             : {@{Id=E7C3B2F0-F3C5-48DF-AF2B-10FED6D72E7A; IsEnabled=False; Name=Microsoft Windows Filtering Platform}, @{Id=E9B59CFA-2BE1-4B21-828F-B6FBDBDDC017; IsEnabled=True;
Name=Microsoft Azure VFP Switch Extension}, @{Id=EA24CD6C-D17A-4348-9190-09F0D5BE83DD; IsEnabled=True; Name=Microsoft NDIS Capture}}
Flags                  : 0
Health                 : @{LastErrorCode=0; LastUpdateTime=132749312994946529}
ID                     : FAD80830-616C-4BC2-8758-BA64AEA8F627
IPv6                   : False
LayeredOn              : F1F2F14B-64E9-4F6A-B7E7-2E528EBEA406
MacPools               : {@{EndMacAddress=00-15-5D-74-EF-FF; StartMacAddress=00-15-5D-74-E0-00}}
ManagementIP           : 10.20.30.11
MaxConcurrentEndpoints : 1
Name                   : Calico
Policies               : {@{Type=HostRoute}, @{DestinationPrefix=100.244.49.64/26; DistributedRouterMacAddress=66-a5-cc-c1-86-97; IsolationId=4096; ProviderAddress=10.20.30.10; Type=RemoteSubnetRoute}}
Resources              : @{AdditionalParams=; AllocationOrder=1; Allocators=System.Object[]; Health=; ID=DE451C81-5986-4BF9-B720-2611F5DEE310; PortOperationTime=0; State=1; SwitchOperationTime=0;
VfpOperationTime=0; parentId=6C549FF2-89EA-4D7E-B5C2-704140569CFB}
State                  : 1
Subnets                : {@{AdditionalParams=; AddressPrefix=100.244.206.64/26; GatewayAddress=100.244.206.65; Health=; ID=D521D64F-8B6E-4E3D-837B-1385B350248B; ObjectType=5; Policies=System.Object[];
State=0}}
TotalEndpoints         : 1
Type                   : Overlay
Version                : 38654705669
```

For the final smoke test, we hit the another IIS pod on `100.244.206.67` this time:

```shell
bash-5.1# curl 100.244.206.67
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1" />
<title>IIS Windows Server</title>
<style type="text/css">
<!--
body {
	color:#000000;
	background-color:#0072C6;
	margin:0;
}

#container {
	margin-left:auto;
	margin-right:auto;
	text-align:center;
	}

a img {
	border:none;
}

-->
</style>
</head>
<body>
<div id="container">
<a href="http://go.microsoft.com/fwlink/?linkid=66138&amp;clcid=0x409"><img src="iisstart.png" alt="IIS" width="960" height="600" /></a>
</div>
</body>
```

Here is a summarization of the Node traffic output:

```
vagrant@controlplane:~$ tshark -i any
   ...
   1 7.592594949 100.244.49.69 → 100.244.206.67 TCP 76 44836 → 80 [SYN] Seq=0 Win=64860 Len=0 MSS=1410 SACK_PERM=1 TSval=2685085784 TSecr=0 WS=128
   2 7.593072350 100.244.206.67 → 100.244.49.69 TCP 68 80 → 44836 [SYN, ACK] Seq=0 Ack=1 Win=65535 Len=0 MSS=1410 WS=256 SACK_PERM=1
   3 7.593126368 100.244.49.69 → 100.244.206.67 TCP 56 44836 → 80 [ACK] Seq=1 Ack=1 Win=64896 Len=0
   4 7.593211096 100.244.49.69 → 100.244.206.67 HTTP 134 GET / HTTP/1.1
```

### Digging deeper

Luckily the issue was clear with a few basic powershell functions, sometimes it is required to dig deeper to
find the root cause, this reference [4] provides a very good step by step flow of debugging, and 
to illustrate, lets bring an interesting example.

Each container has a virtual network adapter (vNIC) which is connected to a Hyper-V virtual switch (vSwitch).
These endpoints can be listed using the [collectlogs.ps1](https://github.com/microsoft/SDN/blob/master/Kubernetes/windows/debug/collectlogs.ps1)
script. Logs and data gathering will be dumped in a few txt files, it was possible to solve our issue
in the HNS network layer, but lets analyze what exists inside the container configuration, from the `endpoint.txt` file
it is possible to find:

1. DNS configuration
2. gateway route configuration
3. Health check
4. IP and MAC addresss
5. The policies for this endpoint, No NAT between pods, vxlan encapsulation for services, etc.

```shell
{
    "ActivityId":  "05E5FC25-2E70-4B8E-A80C-5C1DE2F3CC18",
    ...
    "DNSServerList":  "10.96.0.10",
    "DNSSuffix":  "default.svc.cluster.local,svc.cluster.local,cluster.local",
    ...
    "EncapOverhead":  50,
    "GatewayAddress":  "100.244.206.65",
    "Health":  {
      "LastErrorCode":  0,
      "LastUpdateTime":  132749313131273390
    },
    "ID":  "26DCB6EE-F66B-4AB8-98D2-8223F769894A",
    "IPAddress":  "100.244.206.67",
    "MacAddress":  "0E-2A-64-f4-ce-43",
    "Name":  "a85b1b1b99600bf50f265b595b64ea6d3f952c3449e03565066b3fddda1b8052_Calico",
    "Namespace":  {
      "ID":  "08191D89-ACE1-4E50-AACA-7A2F1D539171",
      "IsDefault":  false
    },
    "Policies":  [
      {
        "ExceptionList":  [
          "10.96.0.0/12",
          "100.244.0.0/16"                       
        ],
        "Type":  "OutBoundNAT"
      },
      {
        "DestinationPrefix":  "10.96.0.0/12",
        "NeedEncap":  true,
        "Type":  "ROUTE"
      },
    ...
    ]
}
```

Other inspections can be made in VFP rules with `vfpctrl`, or using `startpacketcapture.cmd` to extract 
and analyze the traffic with `Message Analyzer`:

![sniff](./images/sniff-ok.png?width=1024px "sniff")

## Help needed

Are you interested to help enhance the development environment? Here goes a list of some open issues: 

- [Add full Antrea CNI support](https://github.com/kubernetes-sigs/sig-windows-dev-tools/issues/80)
- [Explore VMware workstation as alternative](https://github.com/kubernetes-sigs/sig-windows-dev-tools/issues/81)
- [Use containerd as default controlplane runtime](https://github.com/kubernetes-sigs/sig-windows-dev-tools/issues/3)
- [Environment health check](https://github.com/kubernetes-sigs/sig-windows-dev-tools/issues/76)

## References

[1] https://github.com/microsoft/Windows-Containers/projects/1

[2] https://github.com/kubernetes/community/tree/master/sig-windows

[3] https://github.com/kubernetes-sigs/sig-windows-dev-tools/

[4] https://docs.projectcalico.org/networking/vxlan-ipip

[5] https://argonsys.com/microsoft-cloud/library/troubleshooting-kubernetes-networking-on-windows-part-1/

[6] https://www.youtube.com/watch?v=BkBgR2uxFCk&t=1912s
