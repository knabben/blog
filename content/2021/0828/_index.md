+++
title = "CNI plugins and kindnet"
description = "Walkthrough of CNI plugins, IPs and routing created by kindnet"
date = "2021-08-28"
markup = "mmark"
+++

## Introduction

With the advance of Containers, the network segmentation was required at level of the isolated applications, 
logically  it's another copy of the network stack, with it's own routes, firewall rules and net devices. 
By convention these named network namespaces (ns) is an object at  /var/run/netns/<name> as we are going 
to see in the sequence. A few tools to manage netns at user-space level resides on [1].

A standard called CNI (Container Network Interface) was created, to formalize and create a baseline that 
consists of a specification and libraries for writing plugins to configure network interfaces in Linux containers,
along with a number of supported plugins. CNI concerns itself only with network connectivity of containers and 
removing allocated resources when the container is deleted. Because of this focus, CNI has a wide 
range of support and the specification is simple to implement. 

In the post will be examined Kindnet the CNI implementation of Kind [3] with the goal to understand the 
main flows and show a few tools to debug it.

## Network namespace

This blog post resumes pretty well the concept used on network namespace. [4]

Generally speaking, an installation of Linux shares a single set of network interfaces and routing table entries.
...
With network namespaces, you can have different and separate instances of network interfaces and routing 
tables that operate independent of each other.

So, based on [5], we can have a sequence of commands to create a two network namespace called `ns1` and `ns2`
for illustation propose. Create a virtual ethernet PAIR (veth), think of this as a ethernet cable between 
the host machine and the namespace application. Important to note that to link ALL namespaces we need
a new device called BRIDGE, this device has an IP and work as a new gateway.

```shell
# Create two namespaces: ns1 and ns2
$ ip netns add ns1
$ ip netns add ns2

# Create the veth pairs for namespace and for host 
$ ip link add veth10 type veth peer name veth11  # host -> ns1
$ ip link add veth20 type veth peer name veth21  # host -> ns2

# Add the veth pairs to the namespaces
$ ip link set veth11 netns ns1
$ ip link set veth21 netns ns2

# Configure the interfaces in the network namespaces with IP address
$ ip netns exec ns1 ip addr add 172.16.0.2/24 dev veth11  # add ip addr on ns1
$ ip netns exec ns2 ip addr add 172.16.0.3/24 dev veth21  # add ip addr on ns2

# Enabling the interfaces inside the network namespaces
$ ip netns exec ns1 ip link set dev veth11 up
$ ip netns exec ns2 ip link set dev veth21 up

# Create the bridge
$ ip link add name br0 type bridge

# Add the network namespaces interfaces to the bridge
$ ip link set dev veth10 master br0
$ ip link set dev veth20 master br0

# Assigning the IP address to the bridge
$ ip addr add 172.16.0.1/24 dev br0

# Enabling the bridge and interfaces connected to the bridge
$ ip link set dev br0 up
$ ip link set dev veth10 up
$ ip link set dev veth20 up

# "Setting the loopback interfaces in the network namespaces"
$ ip netns exec ns1 ip link set lo up
$ ip netns exec ns2 ip link set lo up

# Setting the default route in the network namespaces via bridge IP
$ ip netns exec ns1 ip route add default via 172.16.0.1 dev veth11
$ ip netns exec ns2 ip route add default via 172.16.0.1 dev veth21
```

It's possible to test connectivity between the hosts with `ip netns exec ns1 ping -c 1 172.16.0.3`

## Kindnet

Now there's a better understand on how it's setup behind the scenes, this is an example 
diagram of a default kind setup installation. The CNI main responsability is to automatize
this network container life cycle, executing a list of CNI plugins pre-registered, the 
ADD/DELETE/CHECK commands are available for each plugin.

![scheme](./images/scheme1.png?width=1024px "scheme1")

This is totally configurable via a sequence of plugins but must follow the actual 
specification format to be parseable by the CRI runtime.

### CNI plugins and routing

As we can see in the image, for Kind a simplistic approach is used with only 3 plugins
and no bridge is involved. Seeing the configuration example for the `kind-worker`, 
the explanation of the plugins will come in the next session:

```shell
root@kind-worker:/# cat /etc/cni/net.d/10-kindnet.conflist
{
	"cniVersion": "0.3.1",
	"name": "kindnet",
	"plugins": [
	{
		"type": "ptp",
		"ipMasq": false,
		"ipam": {
			"type": "host-local",
			"dataDir": "/run/cni-ipam-state",
			"routes": [
				{ "dst": "0.0.0.0/0" }
			],
			"ranges": [
				[ { "subnet": "10.244.2.0/24" } ]
			]
		}
		,
		"mtu": 1500

	},
	{
		"type": "portmap",
		"capabilities": {
			"portMappings": true
		}
	}
	]
}
```

For now note that a new container brings it's pair of virtual interfaces and routing entries with the 
namespace IP are configured as part of the CNI setup. Another interesting route here is the CIDR using the
other node as gateway. 

Basically Kindnet reconcile the nodes at each 10 seconds, for the local node it dumps the configuration
below, otherwise it adds the CIDR routes as cited if it's not present.

#### PTP - Point To Point [6]

The ptp plugin creates a point-to-point link between a container and the host by using a veth device.
One end of the veth pair is placed inside a container and the other end resides on the host. 
The host-local IPAM plugin can be used to allocate an IP address to the container.
The traffic of the container interface will be routed through the interface of the host.

In the ADD command of PTP plugin we have the following sequence of events:

```golang
// run the IPAM plugin and get back the config to apply
r, err := ipam.ExecAdd(conf.IPAM.Type, args.StdinData)
```

IPAM (IP Address Management) loads the `host-local` [7]

Host-local IPAM allocates IPv4 and IPv6 addresses out of a specified address range. 
Optionally, it can include a DNS configuration from a resolv.conf file on the host. It allocates ip
addresses out of a set of address ranges and it stores the state locally on the host filesystem, 
therefore ensuring uniqueness of IP addresses on a single host.

After allocating the IP via IPAM host-local, ptp will enable ip forward for the IP version and:

1. setupContainerVeth

```golang
hostVeth, contVeth0, err := ip.SetupVeth(ifName, mtu, "", hostNS)
...
contVeth, err := net.InterfaceByName(ifName)

// clean default route and add the local routing to the container in the ns

for _, r := range []netlink.Route{
    {
      LinkIndex: contVeth.Index,
      Dst: &net.IPNet{
        IP:   ipc.Gateway,
        Mask: net.CIDRMask(addrBits, addrBits),
      },
      Scope: netlink.SCOPE_LINK,
      Src:   ipc.Address.IP,
    },
    {
      LinkIndex: contVeth.Index,
      Dst: &net.IPNet{
        IP:   ipc.Address.IP.Mask(ipc.Address.Mask),
        Mask: ipc.Address.Mask,
      },
      Scope: netlink.SCOPE_UNIVERSE,
      Gw:    ipc.Gateway,
      Src:   ipc.Address.IP,
    },
  } {
    if err := netlink.RouteAdd(&r); err != nil {
      return fmt.Errorf("failed to add route %v: %v", r, err)
    }
  }
}
```

2. setupHostVeth - set the address to the veth pair of the host:

```golang
veth, err := netlink.LinkByName(vethName)
ipn := &net.IPNet{
  IP:   ipc.Gateway,
  Mask: net.CIDRMask(maskLen, maskLen),
}
addr := &netlink.Addr{IPNet: ipn, Label: ""}
if err = netlink.AddrAdd(veth, addr); err != nil {
  return fmt.Errorf("failed to add IP addr (%#v) to veth: %v", ipn, err)
}
...
ipn = &net.IPNet{
  IP:   ipc.Address.IP,
  Mask: net.CIDRMask(maskLen, maskLen),
}
if err = ip.AddHostRoute(ipn, nil, veth); err != nil && !os.IsExist(err)...
```

3. dnsConfSet to overwrite DNS settings

#### Portmap

This is a post-setup plugin that establishes port forwarding - using iptables,
from the host's network interface(s) to a pod's network interface.

It is intended to be used as a chained CNI plugin, and determines the container
IP from the previous result. If the result includes an IPv6 address, it will
also be configured. (IPTables will not forward cross-family).

### Containerd CRI and CNI

The network plugin is now setup in the container runtime via the CRI,
so the ADD/DELETE happens when a new container is bring to life or is killed.

As detailed in the official architecture diagram of Containerd, you can see
the CNI as part of this process, the configuration and binaries are mounted by the CNI
that must exist in the same node.

![containerd](./images/containerd.png?width=1024px "containerd")

On dockershim, this responsability existed on Kubelet in the past, it's not true anymore 
with deprecation of the dockershim and remote CRI configuration.

### Debugging with CNITool

To finalize lets use cnitool [8] to debug the CNI configuration, in this example a new network 
namespace `nstest` is created, and we add the kindnet into the namespace. 

```shell
root@kind-worker2:/# cnitool
cnitool: Add, check, or remove network interfaces from a network namespace
  cnitool add   <net> <netns>
  cnitool check <net> <netns>
  cnitool del   <net> <netns>
root@kind-worker2:/# ip netns add nstest
root@kind-worker2:/# ls /var/run/netns/
cni-5e28f5d1-3a6d-9f79-8708-da18aff121c3  cni-d58e75e1-060b-4476-aa3e-89c3d2c00742  nstest
root@kind-worker2:/# CNI_PATH=/opt/cni/bin cnitool add kindnet /var/run/netns/nstest
{
    "cniVersion": "0.3.1",
    "interfaces": [
        {
            "name": "veth93960e3a",
            "mac": "b2:d7:0a:52:e5:b7"
        },
        {
            "name": "eth0",
            "mac": "0e:83:a9:f9:5d:2c",
            "sandbox": "/var/run/netns/nstest"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 1,
            "address": "10.244.1.7/24",
            "gateway": "10.244.1.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ],
    "dns": {}
```

As noted in the example a new veth was created with the proper routing, we can test a traceroute to a
 `kind-worker` container from the `nstest` with something like:

```shell
root@kind-worker2:/# ip netns exec nstest traceroute 10.244.2.2
traceroute to 10.244.2.2 (10.244.2.2), 30 hops max, 60 byte packets
 1  10.244.1.1 (10.244.1.1)  0.025 ms  0.006 ms *
 2  kind-worker.kind (172.18.0.3)  0.045 ms  0.019 ms *
 3  10.244.2.2 (10.244.2.2)  0.027 ms * *

root@kind-worker:/# tshark -i veth8dae5680  # from a regular ping
25 12.362370462   10.244.1.7 ? 10.244.2.2   ICMP 98 Echo (ping) request  id=0x576e, seq=5/1280, ttl=62
26 12.362400792   10.244.2.2 ? 10.244.1.7   ICMP 98 Echo (ping) reply    id=0x576e, seq=5/1280, ttl=64 (request in 25)
```

## References

[1] https://man7.org/linux/man-pages/man8/ip-netns.8.html

[2] https://www.cni.dev/

[3] https://kind.sigs.k8s.io/

[4] https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/

[5] https://github.com/kristenjacobs/container-networking/tree/master/1-network-namespace

[6] https://www.cni.dev/plugins/current/main/ptp/

[7] https://www.cni.dev/plugins/current/ipam/host-local/

[8] https://www.cni.dev/docs/cnitool/
