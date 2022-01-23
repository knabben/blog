+++
title = "KPNG Windows Userspace"
description = "KPNG windows userspace backend support"
date = "2022-01-22"
markup = "mmark"
+++

## Introduction

KPNG (KubeProxy Next Generation, kproxy v2) has as a goal provide a more scalable version of service proxies
on Kubernetes, the idea is provide a *core* system that comunicates with the API server and provides an SHIM 
interface for the backends. Backends are specialized code that implement the required steps to provide the 
load balancing for the endpoints based on the data plane technology of choice (iptables, ipvs, HCN, etc.)

In this post there will an integration walkthough on the first Windows backend (userspace) port to KPNG
and will be provided a few ways to use it on [SIG-Windows Development Tooling](#link-1).

Read more about the motivations and ideas behind KPNG in the [KEP](#link-2).

## Compiling KPNG

First thing is to compile kpng executable and the backend as a standalone process. Clone
the repository on your GOPATH.

### kPNG core

On kpng folder, enter the cmd folder and install the windows OS version, on your **GOPATH/bin**
you will find the **kpng.exe** binary.

```shell
kpng$ cd cmd/
kpng/cmd$ GOOS=windows go install ./kpng

~/go/bin$ tree
└── windows_amd64
    └── kpng.exe
```

### kPNG windows backend

The windows userspace is in-tree but the compilation happens as a standalone binary.
It connects to the core kpng via gRPC server.

```shell
kpng$ cd backend/windows/userspace
kpng/backend/windows/userspace$ GOOS=windows go build -o winuserspace.exe ./...
```

### Building a Dockerfile

At this point you have 2 binaries, you can run this directly in the cluster as
a process or as a daemonset, for the later lets create the Dockerfile first.

Use a Windows machine to build the image, a working version can be find on **`knabben/kpng:latest`**


```dockerfile
FROM mcr.microsoft.com/windows/nanoserver:1809

ENV PATH="C:\Program Files\PowerShell;C:\utils;C:\Windows\system32;C:\Windows;"

COPY ./kpng.exe /kpng/kpng.exe
COPY ./winuserspace.exe /kpng/winuserspace.exe

ENTRYPOINT ["PowerShell"]
```

Move both binaries to a folder and add this Dockerfile. You can compile with:

```shell
docker build --isolation=hyperv --pull -t "knabben/kpng:latest" -f .\Dockerfile .
```

## Replacing Kube-proxy

At this point there're two ways to run it, first is adding the binaries as a service
the second is directly on your node using `hostProcess`.

### NSSM (Non-Sucking Service Manager)

In this example we are reusing the kube-proxy configuration from the Antrea CNI installation
in the dev environment.

First we stop the Kube-proxy service (removing the dependency from Antrea-Agent), we add a new
service called kpng with the core binary, and another service with the backend.

Don't forget to add the log parameters.

```powershell
# Remove kube-proxy dependency from antrea Agent
nssm.exe set antrea-agent DependOnService -kube-proxy
nssm.exe stop kube-proxy

$KubeProxyConfig="C:/k/antrea/etc/kube-proxy.conf"

$nssm = (Get-Command nssm).Source

# Install kpng.exe as a service
& nssm install kpng "C:/forked/kpng.exe" "kube to-api --kubeconfig=$KubeProxyConfig"
& nssm set kpng Start SERVICE_DELAYED_AUTO_START
& nssm set kpng AppStdout C:\var\log\kpng\kpng.INFO.log
& nssm set kpng AppStderr C:\var\log\kpng\kpng.ERR.log

# Install winuserspace.exe backend as a service
& nssm install winuserspace "C:/forked/winuserspace.exe" "-v=4"
& nssm set winuserspace DependOnService kpng
& nssm set winuserspace Start SERVICE_DELAYED_AUTO_START
& nssm set winuserspace AppStdout C:\var\log\kpng\winuserspace.INFO.log
& nssm set winuserspace AppStderr C:\var\log\kpng\winuserspace.ERR.log

# Starting both services
nssm start kpng
nssm start winuserspace
```

### [HostProcess](#link-3) 

For hostProcess you need at least containerd 1.6+ and Kubernetes>=1.23 to have the feature as beta on Windows.

```shell
NAME           STATUS   ROLES                  AGE     VERSION                         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                                  KERNEL-VERSION     CONTAINER-RUNTIME
controlplane   Ready    control-plane,master   5h46m   v1.23.3-rc.0.5+eb8a499627d33c   10.20.30.10   <none>        Ubuntu 20.04.3 LTS                        5.4.0-81-generic   containerd://1.5.5
winw1          Ready    <none>                 5h36m   v1.23.3-rc.0.5+eb8a499627d33c   10.20.30.11   <none>        Windows Server 2019 Standard Evaluation   10.0.17763.2452    containerd://1.6.0-rc.1
```

The following is the specification of a daemonset used in the experiment:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: kpng
  name: kpng-windows
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: kpng
  template:
    metadata:
      labels:
        k8s-app: kpng
    spec:
      securityContext:
        windowsOptions:
          hostProcess: true
          runAsUserName: "NT AUTHORITY\\system"
      hostNetwork: true
      containers:
      - image: knabben/kpng:latest
        args: ["$env:CONTAINER_SANDBOX_MOUNT_POINT/kpng/winuserspace.exe", "-v=4"]
        workingDir: "$env:CONTAINER_SANDBOX_MOUNT_POINT/kpng/"
        name: kpng-backend
        imagePullPolicy: Always
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
      - image: knabben/kpng:latest
        args: ["$env:CONTAINER_SANDBOX_MOUNT_POINT/kpng/kpng.exe", "kube to-api --kubeconfig=$env:CONTAINER_SANDBOX_MOUNT_POINT/kpng/etc/kube-proxy.conf"]
        workingDir: "$env:CONTAINER_SANDBOX_MOUNT_POINT/kpng/"
        name: kpng
        imagePullPolicy: Always
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        volumeMounts:
        - mountPath: c:\\kpng\\etc\\
          name: kube-proxy
      volumes:
      - name: kube-proxy
        hostPath:
          path: C:\\k\\antrea\\etc
          type: Directory
      nodeSelector:
        kubernetes.io/os: windows
```

A few notes in the spec, securityContext as hostProcess and runAsUserName are required. The kpng container needs access to the kube-proxy.conf (to watch cluster objects).
Mounting the hostPath volume from the actual kube-proxy.

### Smoke testing

The version of Antrea used on sig-windows-dev-tools is 1.4, and Kube-proxy is used for NodePort only, more details on [Antrea live](#link-4).

In the example we are testing only the last hostProcess step. Showing the pods, note the IP of the pod is the same as 
the windows node and two containers are running inside of it (kpng core and kpng backend).

```shell
$ kubectl -n kube-system get daemonset
NAMESPACE     NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
kube-system   antrea-agent   1         1         1       1            1           kubernetes.io/os=linux     5h56m
kube-system   kpng-windows   1         1         1       1            1           kubernetes.io/os=windows   3h21m

$ kubectl get pods -A -o wide
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
default       netshoot                               1/1     Running   0          3h27m   100.244.0.9   controlplane   <none>           <none>
default       whoami-windows-6cf6f67f56-7npz9        1/1     Running   0          3h27m   100.244.1.4   winw1          <none>           <none>
kube-system   kpng-windows-pd2bs                     2/2     Running   0          3h26m   10.20.30.11   winw1          <none>           <none>
```

To test it lets create a new service of type NodePort, pointing to the pod `whoami-windows-6cf6f67f56-7npz9`:

```shell
$ kubectl get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
whoami-windows   NodePort    10.108.182.129   <none>        80:31134/TCP   3h2
```

Access from another node (10.20.30.10) -> 10.20.30.11:31134:

```shell
vagrant@controlplane$ curl 10.20.30.11:31134
I'm whoami-windows-6cf6f67f56-l24gf running on windows/amd64

IP: fe80::b1ea:1b62:be2e:6fc8
IP: 100.244.1.5
IP: ::1
IP: 127.0.0.1
ENV: ALLUSERSPROFILE=C:\ProgramData
ENV: APPDATA=C:\Users\ContainerUser\AppData\Roaming
```

Finally, inspect the logs from the backend as follow, you will see new connections in the service being forwarded to the endpoint.

```shell
$ kubectl -n kube-system logs kpng-windows-pd2bs -c kpng-backend -f
I0123 12:09:23.591710    3260 roundrobin.go:114] "NextEndpoint for service" servicePortName="default/whoami-windows" address="10.20.30.10:36898" endpoints=[100.244.1.5:8080]
I0123 12:09:23.591710    3260 proxysocket.go:92] "Mapped service to endpoint" service="default/whoami-windows" endpoint="100.244.1.5:8080"
I0123 12:09:23.592550    3260 proxysocket.go:148] "Creating proxy between remote and local addresses" inRemoteAddress="10.20.30.10:36898" inLocalAddress="10.20.30.11:31134" outLocalAddress="100.244.1.1:49560" outRemoteAddress="100.244.1.5:8080"
I0123 12:09:23.593028    3260 proxysocket.go:157] "Copying remote address bytes" direction="to backend" sourceRemoteAddress="10.20.30.10:36898" destinationRemoteAddress="100.244.1.5:8080"
I0123 12:09:23.593028    3260 proxysocket.go:157] "Copying remote address bytes" direction="from backend" sourceRemoteAddress="100.244.1.5:8080" destinationRemoteAddress="10.20.30.10:36898"
I0123 12:09:23.598971    3260 proxysocket.go:164] "Copied remote address bytes" bytes=81 direction="to backend" sourceRemoteAddress="10.20.30.10:36898" destinationRemoteAddress="100.244.1.5:8080"
I0123 12:09:23.598971    3260 proxysocket.go:164] "Copied remote address bytes" bytes=2385 direction="from backend" sourceRemoteAddress="100.244.1.5:8080" destinationRemoteAddress="10.20.30.10:36898"
```

### Listening

{{< youtube rHFlEkaBZIQ >}}


## References


<span id="link-1">[1]</span> https://github.com/kubernetes-sigs/sig-windows-dev-tools/

<span id="link-2">[2]</span> https://github.com/kubernetes/enhancements/pull/2094/

<span id="link-3">[3]</span> https://github.com/kubernetes-sigs/sig-windows-tools/blob/master/hostprocess/calico/kube-proxy/kube-proxy.yml

<span id="link-4">[4]</span> https://www.youtube.com/watch?v=3Z0NOrETjxY
