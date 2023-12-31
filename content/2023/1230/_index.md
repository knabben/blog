+++
title = "Running Windows cluster locally"
description = "Challenges on running Hybrid OS clusters on local machines for development propose."
date = "2023-12-30"
+++

## Introduction

Nowadays, having Hybrid OS Kubernetes clusters is a requirement for any PaaS product. Whether they are Kubernetes managed solutions or not, the capability to run multiple operating systems coexisting in the same Kuberentes cluster has been a reality for quite some time. A few years ago, an initial attempt to create a local cluster in the [SIG-Windows Dev Tools](https://github.com/kubernetes-sigs/sig-windows-dev-tools) subproject showed how challenging it can be due to different requirements, such as multiple host operating systems, and maintaining an immutable base image updated with each new release.

This post discusses some new experiments in the field of virtualization. Qemu with KVM has been proven to be a good solution for running virtual machines with near-native performance, making it a valid technology to test and validate the hypothesis of running a lightweight setup locally. It's worth noting that other projects, such as [K3s](https://github.com/k3s-io/k3s/pull/7259), are also attempting to bring such technology locally. It's important to consider alternative approaches after two years and explore more native alternatives.

{{< mermaid >}}
sequenceDiagram

    participant adm as Admin
    participant kind as Kind
    participant win as QEMU/Windows
    participant calico as Calico CNI
    
    adm->>win: Create a new Windows 2022 VM
    adm->>kind: Create a new Kubernetes cluster on host 
    adm->>+calico: Install the Tigera Operator + CRD
    calico-->>-kind: Bootstrap the daemonset and controllers 
    
    adm->>+win: Kubeadm join with generated token
    win->>-kind: Joins the Windows VM in the Cluster
    calico-->>+calico: Windows node join detected  
    calico->>-win: Run the Windows Node Agent (pod) 
    adm->>kind: Run the E2E test suite    
{{< /mermaid >}}

### QEMU/KVM installations and Windows VMs

Let's skip this in terms of complexity for now, in later posts there'll be deep dives related to the automatic installation of Windows on QEMU/KVM, on this point we part from the principle the final user already have a running version of Windows Datacenter 2019 or 2022. To enable the GUI use the Desktop Experience.

![install1](./images/install1.png?width=600px "install")
![install2](./images/install2.png?width=600px  "install")

Check the interface with the *ipconfig* command in the node:

```shell
Ethernet adapter vEthernet (Ethernet):

Connection-specific DNS Suffix  . :
Description . . . . . . . . . . . : Hyper-V Virtual Ethernet Adapter
   Physical Address. . . . . . . . . : 52-54-00-F5-C6-91
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   Link-local IPv6 Address . . . . . : fe80::2053:8fc5:a377:3f3c%12(Preferred)
   IPv4 Address. . . . . . . . . . . : 192.168.122.220(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : Sunday, December 31, 2023 7:56:23 AM
   Lease Expires . . . . . . . . . . : Sunday, December 31, 2023 9:26:23 AM
   Default Gateway . . . . . . . . . : 192.168.122.1
   DHCP Server . . . . . . . . . . . : 192.168.122.1
   DHCPv6 IAID . . . . . . . . . . . : 206722048
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-2D-23-83-61-52-54-00-F5-C6-91
   DNS Servers . . . . . . . . . . . : 192.168.122.1
   NetBIOS over Tcpip. . . . . . . . : Enabled
                                       PS C:\>
```

Dump the XML from virsh (VM) to show the gateway IP and the DHCP server range, as observed the
current interface has `192.168.122.220`.

```shell
virsh # net-dumpxml default
<network connections='1'>
  <name>default</name>
  <uuid>f15a6c78-1aed-410a-bcf2-a725a9ef04df</uuid>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:b6:05:dd'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.240'/>
    </dhcp>
  </ip>
</network>
```

### Creating a Kind cluster

The Control-plane (and a few Linux workers) are going to exist in the host using KIND, the APIServer uses the KVM/QMEU bridge address *192.168.122.1* to bind the Apiserver and 6443 TCP port, this way
the Windows VM can access the API server and join the node in the cluster

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "192.168.0.0/19"
  disableDefaultCNI: true
  apiServerAddress: "192.168.122.1"
  apiServerPort: 6443
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Run `kind create cluster --config kind.conf` and wait for the command to finish, the important parameter here is the POD subnet *192.168.0.0/19* -- this is going to be used by the workload to allocate the pods, instead of installing [KindNET](https://opssec.in/2021/0828/) the default Kind CNI, Calico is going to be used.

Checking the docker containers running Kubernetes on Linux nodes.

```shell
❯ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                          NAMES
fe9c2bf3f9e5   kindest/node:v1.27.3   "/usr/local/bin/entr…"   49 minutes ago   Up 49 minutes                                  kind-worker2
5800a6a47917   kindest/node:v1.27.3   "/usr/local/bin/entr…"   49 minutes ago   Up 49 minutes   192.168.122.1:6443->6443/tcp   kind-control-plane
2ad0a5579ce3   kindest/node:v1.27.3   "/usr/local/bin/entr…"   49 minutes ago   Up 49 minutes                                  kind-worker
```

### Installing Calico with Operators

All nodes are still in *NotReady* state, due the missing CNI installation, install Calico CNI using the operator, it's a pretty straight forward procedure, using v3.27.0, Windows is not hostProcess supported by default. Start creating the Tigera Operator YAML, and downloading the CR yaml:

```shell
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

$ wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

Change the CR yaml with the default settings for the Kind cluster, disabling BGP and setting 
the CIDR to same Kind POD created. VXLan encapsulation is being used, but BGP is available for Windows a well:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  serviceCIDRs: ["10.96.0.0/12"]
  calicoNetwork:
    bgp: Disabled
    windowsDataplane: "HNS"
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/19
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "192.168.122.1"
  KUBERNETES_SERVICE_PORT: "6443"
```

Wait for the APIServer pod to be up, make sure you set StrictAffinity option in the IPAM Configuration:

```shell
$ kubectl patch ipamconfigurations default --type merge --patch='{"spec": {"strictAffinity": true}}'
```

### Join the Windows node

The Kind Cluster is running at this point, time to bring the Windows VM joined to our cluster,
start installing Containerd, This script is installing latest **1.7.11**, a restart is required
if the Windows Container subsystem was not installed before.

```shell
PS C:\> Invoke-WebRequest https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/Install-Containerd.ps1 
  -OutFile c:\Install-Containerd.ps1

PS C:\> C:\Install-Containerd.ps1 -ContainerDVersion 1.7.11
    -skipHypervisorSupportCheck -CNIConfigPath "c:/etc/cni/net.d" 
    -CNIBinPath "c:/opt/cni/bin"
```

Next prepare the node and install the Kubeadm, Kubelet and Kubectl tooling, a new Windows service *kubelet* will exist:

```shell
PS C:\> Invoke-WebRequest https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/PrepareNode.ps1 -OutFile c:\PrepareNode.ps1

PS C:\> c:\PrepareNode.ps1 -KubernetesVersion v1.27.3
```

The last step after you have set append `c:\Windows\System32\drivers\etc\hosts` with `kind-control-plane 192.168.122.1` is:

```shell
kubeadm join kind-control-plane:6443 --token ejej6v... \
  --discovery-token-ca-cert-hash sha256:a97b5928dd546d13359a31a4dc6fe26a1a...
```

The Calico Windows agent must be running (check if a External HNS net exists with *Get-HNSNetwork* in the Windows host), Kube-proxy must be up as well:

```shell
❯ kubectl get pods -A -w | grep windows
calico-system                    calico-node-windows-mhqt2                    2/2     Running   0               97m
kube-system                      kube-proxy-windows-f7975                     1/1     Running   1 (7m19s ago)   7m29s
```

## Running the Windows E2E

It's possible to run the test suite, compiling from the Kubernetes source if necessary:

```shell
$ cd go/src/k8s.io/kubernetes
$ make WHAT="test/e2e/e2e.test"
$ ./_output/bin/e2e.test --ginkgo.trace --ginkgo.v --node-os-distro windows --kubeconfig ~/.kube/config --provider=local --ginkgo.focus="\[sig-windows\]"
```

![e2e](./images/e2e.png?width=1024px "e2e")

Now, jf you are looking for a full test suite to ensure your cluster is ready for operations take a look in the [Windows Operational Readiness](https://github.com/kubernetes-sigs/windows-operational-readiness) subproject!

## Thoughts on automation

The initial idea is to modify the "immutable" node to a mutable one using the "agent" concept. This would allow end-users to install, modify and update software during runtime or bootstrap time. All of these operations, such as bootstrapping (installing containerd, kubelet binaries and services, changing default settings) could be automated. In projects like Image-builder, these tasks are executed as custom roles through packer provisioning, and there are several golang libraries available to wrap up Ansible and reuse these tasks.

There are alternative options to achieve a similar outcome, such as using golang command lines execution with PowerShell in the node. This requires accessing the node via a pre-installed SSH server with PKI access for the administration user. Once the node is baked, the join procedure in the kind-created cluster in the Linux host takes place. At this point, Docker and KVM/Qemu are required.

Two important things need to be addressed to fully automate the cluster:

1. replicating the procedure for Windows Hyper-V native VMs and
2. Automating the [Windows installation](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/automate-windows-setup) lifecycle across different hypervisors and OSs.

It is crucial to emphasize the significance of providing a smooth experience to the final user or Kubernetes developers while utilizing these environments. This will enable them to focus on the problem they are attempting to solve rather than dealing with unexpected failures in the infrastructure setup. This can be considered one of the primary requisites of the project.

## Contribute

Interested in contribute? Find the team on **Kubernetes Slack** workspace **#sig-windows** channel, show up and say HI!
