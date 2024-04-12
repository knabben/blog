+++
title = "Kubernetes vSphere CSI on Windows"
description = "Tips and ideas on setting up Kubernetes Windows nodes"
date = "2024-02-01"
+++

![tainha](/images/sea/sea7.png?width=230px "tainha")

## Introduction

The purpose of the upcoming posts is to discuss Day 2 operations. Assuming that you have a running cluster, we will explore additional capabilities that can be added to ensure smooth operation and management. In particular, this post will focus on CSI and Storage within vSphere (and Windows), although most of the process can be applied to other clouds.

The Kubernetes v1.13 release introduced the Container Storage Interface (CSI), which replaces the previous "in-tree" volume plugins included in k/k's codebase. With the old architecture, vendors were required to maintain their plugins through the entire Kubernetes release lifecycle. CSI provides a standard interface pattern (along with CNI, CRI, etc.) that allows third-party plugins to exist outside the main Kubernetes repository. The goal of CSI is to provide a standard method for exposing arbitrary block and file storage to containers, allowing these vendors to provide solutions by themselves.

A CSI driver is required for the cluster to utilize underlying infrastructure resources. The vSphere CSI driver is a plugin that sits outside the Kubernetes codebase, allowing containerized workloads to access vSphere storage. This plugin offers support for different types of storage, including vSAN. The vSphere CSI driver communicates with the control plane on the vSphere server for all storage provisioning operations. On Kubernetes, the CSI driver is used with the vSphere CPI (Cloud Provider Interface). The CSI driver is shipped as a container image and must be deployed in the cluster.

In this post, it is important to understand the Cloud Native Storage Server Component on vSphere, also known as the CNS control plane inside the vSphere server, and focus on the CSI and Kubernetes layers. CNS control plane is an extension of the vCenter Server management that implements provisioning and life cycle operations for container volumes. When provisioning container volumes, it interacts with the vCenter Server to create storage objects that back the volumes. The Storage Policy-Based Management functionality guarantees the required level of service to the volumes and provides monitoring and backing storage objects.

### [Kubernetes Volumes Objects](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

From a vSphere VSAN DataStore we can set the following objects for a RWO (per Pod mount)

```shell
$ govc datastore.info
Name:        vsanDatastore
  Path:      /dc0/datastore/vsanDatastore
  Type:      vsan
  URL:       ds:///vmfs/volumes/vsan:5253520081e7f5cb-482ce3096504d5fd/
  Capacity:  1024.0 GB
  Free:      1009.4 GB
```

#### StorageClass

A StorageClass sets the parameters for a particular type or class of storage that can be used to dynamically provision PersistentVolumes. StorageClasses are not associated with a specific namespace, and their name in etcd is determined by ObjectMeta.Name. In Kubernetes, there is no default StorageClass, so it must be created manually. In vSphere, the storagePolicyName parameter is crucial as it allows for the attachment of Storage Policy Based Management (SPBM) to the datastore, providing greater control over container volume granularity.

From `kubectl get sc -o yaml vsan-sc`

```yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsan-sc
provisioner: csi.vsphere.vmware.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  csi.storage.k8s.io/fstype: ext4
  datastoreurl: ds:///vmfs/volumes/vsan:5253520081e7f5cb-482ce3096504d5fd/
```

#### PVC (Persistent Volume Claim)

A *PersistentVolumeClaim* (PVC) is a request made by a user for storage space to claim for a persistent volume in the cluster for a specific Pod. It works like a Pod, but instead of consuming node resources, it consumes PV resources. While Pods can request specific levels of resources like CPU and Memory, PVCs can request specific size and access modes. These access modes determine how the PVC can be mounted, such as:

- *ReadWriteOnce: the volume can be mounted as read-write by a single node, typically block volumes.*
- ReadOnlyMany: read-only be many nodes
- ReadWriteMany; read-write by many nodes and Pods, typically file shares.
- ReadWriteOncePod: read-write by a single Pod

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwo-pvc
spec:
  volumeMode: Block
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
  storageClassName: vsan-sc
```

Create the Pod with the volume pointing to the PVC, so the request can be achieved:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox:1.24
      command: ["/bin/sh", "-c", "while true ; do sleep 2 ; done"]
      volumeDevices:
        - devicePath: /dev/xvda
          name: data
  restartPolicy: Never
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: rwo-pvc
```

Check the PVC status, it must be set to `bound`:

```shell
$ kubectl get pvc 
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rwo-pvc   Bound    pvc-35e6c773-2b21-4408-a046-33f27b7238cb   15Gi       RWO            vsan-sc        110s
```

The CNS on vSphere is going to provide the current mounted volumes in the datastore:

![wait](./images/storage.png?width=1024px "wait")

#### PV (Persistent Volume)

A *PersistentVolume* (PV) is a storage unit in the cluster that is provided by an administrator or dynamically provisioned through StorageClasses. It's a resource in the cluster similar to a node. PVs are volume plugins like Volumes but have a lifecycle that isn't dependent on any individual Pod that uses the PV. This API object captures the storage implementation details, such as NFS, iSCSI, or a cloud-provider-specific storage system.

Persistent Volumes (PVs) can be provisioned in two ways: static and dynamic. In the static approach, the admin creates a fixed number of PVs manually, which are then available for consumption. On the other hand, in the dynamic approach, the cluster may try to provision a volume dynamically for a Persistent Volume Claim (PVC). However, this can only happen if the PVC requests a StorageClass, and the admin has previously created and configured that class for dynamic provisioning to occur. On vSphere persistent volumes map to VMDKs on the datastore, with 2 kinds: First Class Disk (FCD) and Improved Virtual Disks( IVD)

From the dynamic PV the object is created after the PVC is created:

```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-35e6c773-2b21-4408-a046-33f27b7238cb   15Gi       RWO            Delete           Bound    default/rwo-pvc   vsan-sc                 114s
```

The PV will be something like:

```shell
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: csi.vsphere.vmware.com
    volume.kubernetes.io/provisioner-deletion-secret-name: ""
    volume.kubernetes.io/provisioner-deletion-secret-namespace: ""
  finalizers:
  - kubernetes.io/pv-protection
  - external-attacher/csi-vsphere-vmware-com
  name: pvc-35e6c773-2b21-4408-a046-33f27b7238cb
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 15Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: rwo-pvc
    namespace: default
  csi:
    driver: csi.vsphere.vmware.com
    volumeAttributes:
      storage.kubernetes.io/csiProvisionerIdentity: 1706894229053-8081-csi.vsphere.vmware.com
      type: vSphere CNS Block Volume
    volumeHandle: c6dd8286-e961-4bc5-bccb-21af5d454306
  persistentVolumeReclaimPolicy: Delete
  storageClassName: vsan-sc
  volumeMode: Block
status:
  phase: Bound
```

## CSI drivers and Components

In the controller pod specification, containers are responsible for managing the volume in the CNS.

- *csi-attacher*: The external-attacher is a sidecar container that attaches volumes to nodes by calling `ControllerPublish` and `ControllerUnpublish` functions of CSI drivers
- *csi-resizer*: An external resizer sidecar container implements the logic of watching the Kubernetes API for PVC edits, issuing the ControllerExpandVolume RPC call against a CSI endpoint, and updating the PersistentVolume object to reflect the new size.
- *vsphere-csi-controller*: - Core vSphere CNS management
- *vsphere-syncer*: The Syncer is responsible for pushing metadata of PVs, PVCs, and Pods to CNS.
- *csi-provisioner*: Sidecar container that watches Kubernetes PersistentVolumeClaim objects and triggers CreateVolume/DeleteVolume against a CSI endpoint
- *csi-snapshotter*: Sidecar container that watches Kubernetes Snapshot CRD objects and triggers CreateSnapshot/DeleteSnapshot against a CSI endpoint.

On each node there’s a local host driver and driver register:

- *node-driver-registrar*: Sidecar container that registers a CSI driver with the kubelet using the kubelet plugin registration mechanism
- *vsphere-csi-node*: The CSI plugin is responsible for provisioning volumes, attaching and detaching them to VMs, mounting and formatting them, and unmounting volumes from the pod within the node.

The vSphere Storage Controller provides an interface for the container orchestrator to manage the lifecycle of vSphere volumes. It allows the creation, expansion, and deletion of volumes and the attachment and detachment of volumes to Node VMs. The vSphere Container Storage Node allows for the formatting and mounting of volumes to the node, using bind mounts for the volumes inside the pod. Before detaching the volume, the node component helps to unmount the volume from the node. Finally, the Syncer pushes PV, VPC, and pod metadata to CVS. This data assists vSphere administrators in determining which Kubernetes clusters, apps, pods, etc. are using the volume. Full sync keeps the CNS up-to-date with Kubernetes volume metadata information.

Installing the drivers on your Kubernetes cluster instructions are provided on the [official documentation page]( https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/3.0/vmware-vsphere-csp-getting-started/GUID-A1982536-F741-4614-A6F2-ADEE21AA4588.html).


## Windows CSI Drivers

The Windows installation has a few subtle distinctions (as usual). After installing CSI on Linux and letting the core components run, the next step is to install the CSI-proxy on Windows. The binary is not available, it’s required to build first and push to the Windows node:

```shell
$ git clone https://github.com/kubernetes-csi/csi-proxy
$ sudo make build
$ ls bin/
csi-proxy.exe
```

To [install the binary](https://github.com/kubernetes-csi/csi-proxy?tab=readme-ov-file#installation) as a service install it with the command (the `csi-proxy.exe` is hosted on `c:\etc\kubernetes\node\bin\csi.proxy.exe`):

```shell
$flags = "-windows-service -log_file=C:\etc\kubernetes\logs\csi-proxy.log -logtostderr=false"
sc.exe create csiproxy start= "auto" binPath= "C:\etc\kubernetes\node\bin\csi-proxy.exe $flags"
sc.exe failure csiproxy reset= 0 actions= restart/10000
sc.exe start csiproxy
```

The CSI proxy is a binary responsible for exposing the gRPC APIs around local storage operations for Windows. The CSI Driver installed from the DaemonSet accesses the socket pipeline from CSI proxy and invokes the operations (a few APIs like Disk, Volume, SMB, FS are available)

After the CSI proxy installation, apply the vSphere CSI Node daemonset for the Windows nodes:

```
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: vsphere-csi-node-windows
  namespace: vmware-system-csi
spec:
  selector:
    matchLabels:
      app: vsphere-csi-node-windows
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: vsphere-csi-node-windows
        role: vsphere-csi-windows
    spec:
      priorityClassName: system-node-critical
      nodeSelector:
        kubernetes.io/os: windows
      serviceAccountName: vsphere-csi-node
      containers:
        - name: node-driver-registrar
          image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.8.0
          args:
            - "--v=5"
            - "--csi-address=$(ADDRESS)"
            - "--kubelet-registration-path=$(DRIVER_REG_SOCK_PATH)"
          env:
            - name: ADDRESS
              value: 'unix://C:\\csi\\csi.sock'
            - name: DRIVER_REG_SOCK_PATH
              value: 'C:\\var\\lib\\kubelet\\plugins\\csi.vsphere.vmware.com\\csi.sock'
          volumeMounts:
            - name: plugin-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration
          livenessProbe:
            exec:
              command:
              - /csi-node-driver-registrar.exe
              - --kubelet-registration-path=C:\\var\\lib\\kubelet\\plugins\\csi.vsphere.vmware.com\\csi.sock
              - --mode=kubelet-registration-probe
            initialDelaySeconds: 3
        - name: vsphere-csi-node
          image: gcr.io/cloud-provider-vsphere/csi/release/driver:v3.0.1
          args:
            - "--fss-name=internal-feature-states.csi.vsphere.vmware.com"
            - "--fss-namespace=$(CSI_NAMESPACE)"
          imagePullPolicy: "Always"
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName          
            - name: CSI_ENDPOINT
              value: 'unix://C:\\csi\\csi.sock'
            - name: MAX_VOLUMES_PER_NODE
              value: "59" # Maximum number of volumes that controller can publish to the node. If value is not set or zero Kubernetes decide how many volumes can be published by the controller to the node.
            - name: X_CSI_MODE
              value: node
            - name: X_CSI_SPEC_REQ_VALIDATION
              value: 'false'
            - name: X_CSI_SPEC_DISABLE_LEN_CHECK
              value: "true"
            - name: LOGGER_LEVEL
              value: "PRODUCTION" # Options: DEVELOPMENT, PRODUCTION
            - name: X_CSI_LOG_LEVEL
              value: DEBUG
            - name: CSI_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODEGETINFO_WATCH_TIMEOUT_MINUTES
              value: "1"
          volumeMounts:
            - name: plugin-dir
              mountPath: 'C:\csi'
            - name: pods-mount-dir
              mountPath: 'C:\var\lib\kubelet'   
            - name: csi-proxy-volume-v1
              mountPath: \\.\pipe\csi-proxy-volume-v1
            - name: csi-proxy-filesystem-v1
              mountPath: \\.\pipe\csi-proxy-filesystem-v1
            - name: csi-proxy-disk-v1
              mountPath: \\.\pipe\csi-proxy-disk-v1    
            - name: csi-proxy-system-v1alpha1
              mountPath: \\.\pipe\csi-proxy-system-v1alpha1
          ports:
            - name: healthz
              containerPort: 9808
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /healthz
              port: healthz
            initialDelaySeconds: 10
            timeoutSeconds: 5
            periodSeconds: 5
            failureThreshold: 3
        - name: liveness-probe
          image: registry.k8s.io/sig-storage/livenessprobe:v2.10.0
          args:
            - "--v=4"
            - "--csi-address=/csi/csi.sock"
          volumeMounts:
            - name: plugin-dir
              mountPath: /csi
      volumes:
        - name: registration-dir
          hostPath:
            path: 'C:\var\lib\kubelet\plugins_registry\'
            type: Directory
        - name: plugin-dir
          hostPath:
            path: 'C:\var\lib\kubelet\plugins\csi.vsphere.vmware.com\'
            type: DirectoryOrCreate
        - name: pods-mount-dir
          hostPath:
            path: \var\lib\kubelet
            type: Directory
        - name: csi-proxy-disk-v1
          hostPath:
            path: \\.\pipe\csi-proxy-disk-v1
            type: ''
        - name: csi-proxy-volume-v1
          hostPath:
            path: \\.\pipe\csi-proxy-volume-v1
            type: ''
        - name: csi-proxy-filesystem-v1
          hostPath:
            path: \\.\pipe\csi-proxy-filesystem-v1
            type: ''
        - name: csi-proxy-system-v1alpha1
          hostPath:
            path: \\.\pipe\csi-proxy-system-v1alpha1
            type: ''
      tolerations:
        - effect: NoExecute
          operator: Exists
        - effect: NoSchedule
          operator: Exists
```
#### Troubleshooting

If after installing the CSI driver, the gRPC socket is getting reset when installing the Kubelet plugin from the node-driver-registrar:

```shell
2024-02-04T05:40:54.2393408-08:00 stderr F {"level":"error","time":"2024-02-04T05:40:54.238813-08:00","caller":"osutils/windows_os_utils.go:509","msg":"csi plugin started on windows node without enabling feature switch","TraceId":"a719f958-
b149-4d92-b432-8dc8cfb4b4a4","stacktrace":"[sigs.k8s.io/vsphere-csi-driver/v3/pkg/csi/service/osutils.(*OsUtils](http://sigs.k8s.io/vsphere-csi-driver/v3/pkg/csi/service/osutils.(*OsUtils)).ShouldContinue\n\t/build/pkg/csi/service/osutils/windows_os_utils.go:509\[nsigs.k8s.io/vsphere-csi-driver/v3/pkg/csi/service.(*vs](http://nsigs.k8s.io/vsphere-csi-driver/v3/pkg/csi/service.(*vs)
phereCSIDriver).NodeGetInfo\n\t/build/pkg/csi/service/node.go:340\[ngithub.com/container-storage-interface/spec/lib/go/csi._Node_NodeGetInfo_Handler\n\t/go/pkg/mod/github.com/container-storage-interface/spec@v1.7.0/lib/go/csi/csi.pb.go:6231\](http://ngithub.com/container-storage-interface/spec/lib/go/csi._Node_NodeGetInfo_Handler%5Cn%5Ct/go/pkg/mod/github.com/container-storage-interface/spec@v1.7.0/lib/go/csi/csi.pb.go:6231%5C)[ngoogle.golang.org/grpc.(*Server](http://ngoogle.golang.org/grpc.(*Server)).processUnaryRPC\n\t/go/pkg/mod/google.golang.org/grpc@v1.47.0/server.go:1283\[ngoogle.golang.org/grpc.(*Server](http://ngoogle.golang.org/grpc.(*Server)).handleStream\n\t/go/pkg/mod/google.golang.org/grpc@v1.47.0/server.go:1620\[ngoogle.golang.org/gr](http://ngoogle.golang.org/gr)
pc.(*Server).serveStreams.func1.2\n\t/go/pkg/mod/google.golang.org/grpc@v1.47.0/server.go:922"}
```

The Windows feature gate must be enabled in the CSI configuration configmap: `internal-feature-states.csi.vsphere.vmware.com`:

```shell
$ kubectl edit configmap -n vmware-system-csi [internal-feature-states.csi.vsphere.vmware.com](http://internal-feature-states.csi.vsphere.vmware.com/)

data:
  csi-windows-support: "true"
```

#### Using a SMB Driver

For SMB installation and usage check Jaime Gonzales [documentation and nodes repository](https://github.com/jaimegag/tkg-zone/tree/main/smb-csi), great source of Windows content!

## Migrating PVC across storages

The first open source tool around to migrate PVCs (and not related to vSphere) is the [pv-migrate](https://github.com/utkuozdemir/pv-migrate), the usage is simple as:

```shell
pv-migrate migrate \
    --source-namespace default \
    --dest-namespace backup \
    old-pvc new-pvc
```

### Using [VolumeSnapshot](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/3.0/vmware-vsphere-csp-getting-started/GUID-E0B41C69-7EEB-450F-A73D-5FD2FF39E891.html)

The official solution is the usage of Volume Snapshot, install it using `bash deploy-csi-snapshot-components.sh`.
After The CRD are installed it's possible to backup the PVC (first create VolumeSnapshotClass (for example vs-class))

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  finalizers:
  - snapshot.storage.kubernetes.io/volumesnapshot-as-source-protection
  - snapshot.storage.kubernetes.io/volumesnapshot-bound-protection
  name: pvc-snapshot
  namespace: default
spec:
  source:
    persistentVolumeClaimName: windows-pvc
  volumeSnapshotClassName: vs-class
```

The restore is simple, create a PVC and point the datasource to the VolumeSnapshot just created:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-restore
spec:
  storageClassName: vs-class
  dataSource:
    name: pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

You will endup with another PVC with the same configuration (and content) in the new PVC created:

```shell
kubo@MJvjY0ETPFqph:~$ kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
windows-pvc   Bound    pvc-2916cdff-f5ca-4d0b-ae97-1abe9ae83f0f   5Gi        RWO            windows        85m
pvc-restore           Bound    pvc-560324d0-719a-4cb0-bc7c-5181d544bb8f   5Gi        RWO            windows        5s
```
