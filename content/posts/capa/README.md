+++
title = "Cluster API Provider for AWS (CAPA)"
description = "Breaking a bunch of controllers on a saturday"
date = "2022-07-12"
markup = "mmark"
+++

## Introduction
 
The Cluster API is a Kubernetes project to bring declarative, Kubernetes-style APIs to cluster creation, configuration, and management. 
You can create new workload clusters from a management cluster in a descritive way, the ClusterAPI controllers provide the initial management cluster 
setup and a tool called *clusterctl* to manage them. Workloads clusters are specilized controllers on separated projects called providers. Each cloud provider will have it's own controllers and ways to build the workload cluster obeying the rules and logic defined in the spec and workflow diagrams here [2]. 

This post in particular will help in a mental mode creation of the Cluster Infrastructure object [3], for the sake of curiosity lets try to find 
out all networking, ELB, SG and firewall rules are created by the cluster controller and understand how external requests go through the apiserver in the cp.

## Management Cluster

From the official CAPI docs - The cluster where one or more Infrastructure Providers run, and where resources (e.g. Machines) are stored. 
Typically referred to when you are provisioning multiple workload clusters. To create the management cluster you need to use your credentials 
and indicate the infrastructure type as an argument.

The management cluster is required to create new clusters and store the configuration of them, think like a main cluster, for the workload clusters.
This cluster will contain all controllers and objects (Machines, Clusters, KubeadmTemplates) to setup the workload clusters.

### Managed vs Unmanaged clusters

A quick note about both types of clusters provided by CAPI. 

Managed clusters are cloud providers services offerings like GKE, EKS and AKS. You don't have access to the control plane and don't need to worry with 
the underlying management of the cluster, the cloud provider manages it for you, you are still responsible to managed things like node pools and initial setup, 
on this post we are going to take a look in particular in the EKS and CAPA managed support. Cloud providers on this mode use their own CNI [4]
 that's integrated with their cloud services (VPC).

Unmanaged clusters as you have guessed are clusters where the customer setup by themselves normally with kubeadm, and you have the total control 
of the cp and nodes. CAPI has support for it as well is all providers, on AWS it setup the nodes using EC2 directly, the mode is defined
in the specs.

### Creating a new management cluster

 After downloading the clusterctl, set the shell variables used by clusterctl to initialize your assets.

```shell
export CLUSTER_TOPOLOGY=true
export AWS_REGION=us-east-1 # This is used to help encode your environment variables
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
export AWS_SESSION_TOKEN=<session-token> # If you are using Multi-Factor Auth.

$ clusterawsadm bootstrap iam create-cloudformation-stack
```

On this phase Attempting to create AWS CloudFormation stack `cluster-api-provider-aws-sigs-k8s-io`, following resources are in the stack:

+---------------------------+--------------------------------------------------------------------------------------+----------------|
| Resource                  |Type                                                                                  |Status          |
+---------------------------+--------------------------------------------------------------------------------------+----------------|
| AWS::IAM::InstanceProfile |control-plane.cluster-api-provider-aws.sigs.k8s.io                                    |CREATE_COMPLETE |
| AWS::IAM::InstanceProfile |controllers.cluster-api-provider-aws.sigs.k8s.io                                      |CREATE_COMPLETE |
| AWS::IAM::InstanceProfile |nodes.cluster-api-provider-aws.sigs.k8s.io                                            |CREATE_COMPLETE | 
| AWS::IAM::ManagedPolicy   |arn:aws:iam::139642426024:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io   |CREATE_COMPLETE | 
| AWS::IAM::ManagedPolicy   |arn:aws:iam::139642426024:policy/nodes.cluster-api-provider-aws.sigs.k8s.io           |CREATE_COMPLETE | 
| AWS::IAM::ManagedPolicy   |arn:aws:iam::139642426024:policy/controllers.cluster-api-provider-aws.sigs.k8s.io     |CREATE_COMPLETE | 
| AWS::IAM::ManagedPolicy   |arn:aws:iam::139642426024:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE | 
| AWS::IAM::Role            |control-plane.cluster-api-provider-aws.sigs.k8s.io                                    |CREATE_COMPLETE |
| AWS::IAM::Role            |controllers.cluster-api-provider-aws.sigs.k8s.io                                      |CREATE_COMPLETE | 
| AWS::IAM::Role            |eks-controlplane.cluster-api-provider-aws.sigs.k8s.io                                 |CREATE_COMPLETE | 
| AWS::IAM::Role            |nodes.cluster-api-provider-aws.sigs.k8s.io                                            |CREATE_COMPLETE |
+---------------------------+--------------------------------------------------------------------------------------+----------------|

```shell
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
$ cslusterctl init --infrastructure aws
```

At this point you have a management cluster with the following pods and namespaces:

```shell
❯ kubectl get pods -A
NAMESPACE                           NAME                                                             READY   STATUS    RESTARTS        AGE
capa-system                         capa-controller-manager-6cd466656-plgbg                          1/1     Running   0               5h8m
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-7dcd76b574-7kshh       1/1     Running   0               5h8m
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-654cbff89f-z84sr   1/1     Running   0               5h8m
capi-system                         capi-controller-manager-7bbc6f959-k7l7m                          1/1     Running   0               5h8m
```

CAPA controller is responsible to match the infrastructure and a few bootstrap group resources through the `capa-controller-manager`.

```shell
eksconfigs                        eksc         bootstrap.cluster.x-k8s.io/v1beta1        true         EKSConfig
eksconfigtemplates                eksct        bootstrap.cluster.x-k8s.io/v1beta1        true         EKSConfigTemplate
awsmanagedcontrolplanes           awsmcp       controlplane.cluster.x-k8s.io/v1beta1     true         AWSManagedControlPlane
awsclustercontrolleridentities    awsci        infrastructure.cluster.x-k8s.io/v1beta1   false        AWSClusterControllerIdentity
awsclusterroleidentities          awsri        infrastructure.cluster.x-k8s.io/v1beta1   false        AWSClusterRoleIdentity
awsclusters                       awsc         infrastructure.cluster.x-k8s.io/v1beta1   true         AWSCluster
awsclusterstaticidentities        awssi        infrastructure.cluster.x-k8s.io/v1beta1   false        AWSClusterStaticIdentity
awsclustertemplates               awsct        infrastructure.cluster.x-k8s.io/v1beta1   true         AWSClusterTemplate
awsfargateprofiles                awsfp        infrastructure.cluster.x-k8s.io/v1beta1   true         AWSFargateProfile
awsmachinepools                   awsmp        infrastructure.cluster.x-k8s.io/v1beta1   true         AWSMachinePool
awsmachines                       awsm         infrastructure.cluster.x-k8s.io/v1beta1   true         AWSMachine
awsmachinetemplates               awsmt        infrastructure.cluster.x-k8s.io/v1beta1   true         AWSMachineTemplate
awsmanagedmachinepools            awsmmp       infrastructure.cluster.x-k8s.io/v1beta1   true         AWSManagedMachinePool
```

### CAPA Controller AWS services

Cluster objects on a managed cluster EKS have `AWSManagedControlPlane` objects for both controlplaneRef and infrastructureRef

```yaml
kind: Cluster
spec:
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: AWSManagedControlPlane
    name: capi-eks-control-plane
  infrastructureRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: AWSManagedControlPlane
    name: capi-eks-control-plane
```

For an unmanaged cluster the controlplaneref object is a `KubeadmControlPlane` and the infrastructure is `AWSCluster`

```yaml
kind: Cluster
spec:

  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: capi-quickstart-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: capi-quickstart
```

Here there's a sequence diagram for the `AWSManagedControlPlane` controller and how it setups the cluster:

{{< img resizedURL="/posts/capa/diagram1.png" originalURL="./diagram1.png" style="width: 100%; " >}}

### Network assets

This diagram shows the created assets in the cloud.

{{< img resizedURL="/posts/capa/diagram2.jpg" originalURL="./diagram2.jpg" style="width: 95%; " >}}

## AWS EKS Workload Cluster

Your machine deployment is based in the generated `AWSMachineTemplate` and this will create the EC2 machines of your choice
and use them as nodes for your EKS cluster.

To create a new workload cluster run the clusterctl command and apply the final YAML.

```
$ clusterctl generate cluster capi-eks-quickstart --kubernetes-version v1.22.9 --worker-machine-count=3 > capi-quickstart.yaml
$ kubectl apply -f capi-quickstart.yaml

$ clusterctl get kubeconfig capi-eks-quickstart > config.yaml
$ export KUBECONFIG=config.yaml

❯ kubectl get pods -A -o wide
NAMESPACE     NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE                          NOMINATED NODE   READINESS GATES
kube-system   aws-node-5f9fk            1/1     Running   0          6h10m   10.0.75.156   ip-10-0-75-156.ec2.internal   <none>           <none>
kube-system   aws-node-gbn2k            1/1     Running   0          6h11m   10.0.92.67    ip-10-0-92-67.ec2.internal    <none>           <none>
kube-system   aws-node-vfggn            1/1     Running   0          6h11m   10.0.124.60   ip-10-0-124-60.ec2.internal   <none>           <none>
kube-system   coredns-7f5998f4c-6mj58   1/1     Running   0          6h21m   10.0.83.111   ip-10-0-92-67.ec2.internal    <none>           <none>
kube-system   coredns-7f5998f4c-v4fbz   1/1     Running   0          6h21m   10.0.90.36    ip-10-0-92-67.ec2.internal    <none>           <none>
kube-system   kube-proxy-blp6m          1/1     Running   0          6h11m   10.0.92.67    ip-10-0-92-67.ec2.internal    <none>           <none>
kube-system   kube-proxy-rwg69          1/1     Running   0          6h11m   10.0.124.60   ip-10-0-124-60.ec2.internal   <none>           <none>
kube-system   kube-proxy-xsgrl          1/1     Running   0          6h10m   10.0.75.156   ip-10-0-75-156.ec2.internal   <none>           <none>
```

You can see the CNI, CoreDNS and Kube-Proxy running in the workload cluster. We have 16382 IP addresses on subnet at AZ `us-east-1a` and they
are allocated to the pods, the AWS CNI uses jumbo frames with 9001 MTU.

### Running tilt 

In the development mode the process is similar to running normally [5] the only difference is that we are running the pods with Tilt,
it will help to reload changes in the kind cluster faster. Create a `tilt-provider.json` file in the cluster-api folder on your GOPATH, 
pointing to the CAPA repository. Enable debug on 30000.

```json
{
  "enable_providers": ["aws","kubeadm-bootstrap", "kubeadm-control-plane"],
  "provider_repos": ["../cluster-api-provider-aws"],
  "default_registry": "gcr.io/aws-bucket",
  "debug": { "core":  {"port": 30000 } },
 ...
}
```

Uninstall the controllers initialized in the cluster and run tilt.

```shell
$ kubectl delete cluster capi-eks-quickstart
$ clusterctl  delete --all 
$ tilt up -v
```

This is the CAPA controller running through TILT.

{{< img resizedURL="/posts/capa/tilt.png" originalURL="./tilt.png" style="width: 100%; " >}}

Try to apply the cluster again and connect a remote delve from your Goland instance. [6]

{{< img resizedURL="/posts/capa/goland.png" originalURL="./goland.png" style="width: 100%; " >}}

### Conclusion

Managed clusters and multi-cloud are the present, and ClusterAPI is powerful enough to provide the enduser and devops engineer
all the tools and controllers to use it, this post is only the tip of the iceberg, if you are interested in the project
check [7] for in deep sessions in the code base.

### Listening

{{< youtube 08Vt4VkDH4A >}}

## References 

[1] https://kubernetes.io/blog/2020/04/21/cluster-api-v1alpha3-delivers-new-features-and-an-improved-user-experience/

[2] https://cluster-api.sigs.k8s.io/developer/providers/implementers.html

[3] https://cluster-api.sigs.k8s.io/developer/providers/cluster-infrastructure.html

[4] https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html

[5] https://cluster-api-aws.sigs.k8s.io/development/tilt-setup.htm

[6] https://www.jetbrains.com/help/go/attach-to-running-go-processes-with-debugger.html#step-2-build-the-application

[7] https://www.youtube.com/watch?v=78BMdEiS43E