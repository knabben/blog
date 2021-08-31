+++
title = "Windows and Calico overlay networking"
description = "Settings and configurations required for CNI and Windows"
date = "2021-08-28"
markup = "mmark"
+++

## Introduction


Windows vs linux network namespace 

Network namespace | tcpip compartments
Bridge and ip routing | vswitch
ip links | vnics and vswitch vfp ports
ipvs and iptables | firewall and vfp rules

### Overlay networking

VXLAN overlay and Flannel/Calico 


An overlay network allows network devices to communicate across an underlying network (referred to as the underlay) without the underlay network having any knowledge of the devices connected to the overlay network.

From the point of view of the devices connected to the overlay network, it looks just like a normal network. 

There are many different kinds of overlay networks that use different protocols to make this happen, but in general they share the same common characteristic of taking a network packet, referred to as the inner packet, and encapsulating it inside an outer network packet. In this way the underlay sees the outer packets without needing to understand how to handle the inner packets.

How the overlay knows where to send packets varies by overlay type and the protocols they use. Similarly exactly how the packet is wrapped varies between different overlay types. In the case of VXLAN for example, the inner packet is wrapped and sent as UDP in the outer packet.

https://docs.projectcalico.org/about/about-networking

VXLan protocol, example debug on tshark

### Felix setup

....

vxlanMode field (VXLAN encapsulation)

### Vagrantfile and multiple interfaces

HNS-Setnetwork - what is the point and where it makes

How to pick the correct interface and why, what is changed
why VXLAN needs to be created upon on it.

### Connectivity tests and debugging

tshark


## References