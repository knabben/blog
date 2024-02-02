+++
title = "Day 2 - Kubernetes Container Storage Interface (on Windows & vSphere)"
description = "Tips and ideas on setting up Kubernetes Windows nodes"
date = "2024-02-04"
+++

### Introduction

The purpose of the upcoming posts is to discuss Day 2 operations. Assuming that you have a running cluster, we will explore additional capabilities that can be added to ensure smooth operation and management. In particular, this post will focus on CSI and Storage within vSphere (and Windows), although most of the process can be applied to other clouds.

The Kubernetes v1.13 release introduced the Container Storage Interface (CSI), which replaces the previous "in-tree" volume plugins included in k/k's codebase. With the old architecture, vendors were required to maintain their plugins through the entire Kubernetes release lifecycle. CSI provides a standard interface pattern (along with CNI, CRI, etc.) that allows third-party plugins to exist outside the main Kubernetes repository. The goal of CSI is to provide a standard method for exposing arbitrary block and file storage to containers, allowing these vendors to provide solutions by themselves.

A CSI driver is required for the cluster to utilize underlying infrastructure resources. The vSphere CSI driver is a plugin that sits outside the Kubernetes codebase, allowing containerized workloads to access vSphere storage. This plugin offers support for different types of storage, including vSAN. The vSphere CSI driver communicates with the control plane on the vSphere server for all storage provisioning operations. On Kubernetes, the CSI driver is used with the vSphere CPI (Cloud Provider Interface). The CSI driver is shipped as a container image and must be deployed in the cluster.

In this post, it is important to understand the Cloud Native Storage Server Component on vSphere, also known as the CNS control plane inside the vSphere server, and focus on the CSI and Kubernetes layers. CNS control plane is an extension of the vCenter Server management that implements provisioning and life cycle operations for container volumes. When provisioning container volumes, it interacts with the vCenter Server to create storage objects that back the volumes. The Storage Policy-Based Management functionality guarantees the required level of service to the volumes and provides monitoring and backing storage objects.

### [Kubernetes Volumes Objects](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

— asciinema - PV,PVC,StorageClass volume mount example, govc disk analysis

#### PV (Persistent Volume)

A *PersistentVolume* (PV) is a storage unit in the cluster that is provided by an administrator or dynamically provisioned through StorageClasses. It's a resource in the cluster similar to a node. PVs are volume plugins like Volumes but have a lifecycle that isn't dependent on any individual Pod that uses the PV. This API object captures the storage implementation details, such as NFS, iSCSI, or a cloud-provider-specific storage system.

Persistent Volumes (PVs) can be provisioned in two ways: static and dynamic. In the static approach, the admin creates a fixed number of PVs manually, which are then available for consumption. On the other hand, in the dynamic approach, the cluster may try to provision a volume dynamically for a Persistent Volume Claim (PVC). However, this can only happen if the PVC requests a StorageClass, and the admin has previously created and configured that class for dynamic provisioning to occur. On vSphere persistent volumes map to VMDKs on the datastore, with 2 kinds: First Class Disk (FCD) and Improved Virtual Disks( IVD)

#### PVC (Persistent Volume Claim)

A *PersistentVolumeClaim* (PVC) is a request made by a user for storage space to claim for a persistent volume in the cluster for a specific Pod. It works like a Pod, but instead of consuming node resources, it consumes PV resources. While Pods can request specific levels of resources like CPU and Memory, PVCs can request specific size and access modes. These access modes determine how the PVC can be mounted, such as:

- *ReadWriteOnce: the volume can be mounted as read-write by a single node, typically block volumes.*
- ReadOnlyMany: read-only be many nodes
- ReadWriteMany; read-write by many nodes and Pods, typically file shares.
- ReadWriteOncePod: read-write by a single Pod

#### StorageClass

A StorageClass sets the parameters for a particular type or class of storage that can be used to dynamically provision PersistentVolumes. StorageClasses are not associated with a specific namespace, and their name in etcd is determined by ObjectMeta.Name. In Kubernetes, there is no default StorageClass, so it must be created manually. In vSphere, the storagePolicyName parameter is crucial as it allows for the attachment of Storage Policy Based Management (SPBM) to the datastore, providing greater control over container volume granularity.



