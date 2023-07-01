+++
title = "Ztunnel on Ambient Mesh"
description = "Debugging Ztunnel on Ambient mesh"
date = "2023-06-30"
+++

## Introduction

There was rewrite of the Layer 4 processing of Istio in [rust](https://istio.io/latest/blog/2023/rust-based-ztunnel/)
This simplification in conjuntion with `Waypoint` Layer 7 made possible to decouple both capabilities
allowing a more robust and straightfoward and faster implementation of a reverse proxy for 
sidecar removal, it is called ztunnel, and that's the proxy, that offer a basic authorization
layer 4 based on SPIFFEID node identification and encryption across services on a very transparent way.

After setting the ambient label in the default and listing the workloads:

```shell
$ kubectl label ns default istio.io/dataplane-mode=ambient
$ kubectl get pods

appa-bf7c45dcb-lmtg2                  1/1     Running   0          4m29s   10.244.1.8    ambient-worker    <none>           <none>
appb-76b56f7cb4-qkjl7                 1/1     Running   0          4m29s   10.244.2.4    ambient-worker2   <none>           <none>
appb-istio-waypoint-6f8dfd8d4-565jx   1/1     Running   0          3m58s   10.244.1.10   ambient-worker    <none>           <none>
```


### Architecture

{{<mermaid align="center">}}
flowchart LR;
    subgraph nodea[ambient-worker]
        appa-->istioout
        istioout-->|Geneve|15001a
        subgraph ztunnela[ztunnel]
        15001a[15001-pistioout]
        end
    end

    15001a-->|HBONE mTLS|istioin

    subgraph nodeb[ambient-worker2]
        istioin-->|Geneve|15008b
        15008b-->appb
        subgraph ztunnelb[ztunnel]
        15008b[15008-pistioin]
        end
        
    end

  classDef ctrl fill:#1568ca,color:white,stroke-width:1px,stroke:#dbffff
  classDef normal fill:#007cff,color:white,stroke-width:1px,stroke:#
  classDef cluster fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;
  classDef cluster fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;

  class appa,appb ctrl;
  class istioout,istioin cluster; 
  class 15008b,15001a normal;
{{</mermaid>}}

This is the flow when the `appa` sends a request to `appb`.

Both IP addresses are part of the `ztunnel-pods-ips`, the packets from them are mark as `0x100/0x100`
and as seen the traffic is sent via Geneve tunnel to `192.168.127.2` via `101` table, the TPROXY
rules captures everything on this interface of the ztunnel and TPROXY (keeping the source IP) directly
to localhost TCP `15001` where the ztunnel is listenning.

The destination is in another host where it receives the HBONE encapsuled packet on 15008 (pistioin), 
with the actual destination still being the pod in the mesh (`appb`). To read more about the [HBONE](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/upgrades#tunneling-tcp-over-http) tunnel.

For same node, we just access it directly rather than making a full network connection.
We *could* apply this to all traffic, rather than just for destinations that are "captured"
However, we would then get inconsistent behavior where only node-local pods have RBAC enforced.

```rust
  if req.request_type == RequestType::DirectLocal && can_fastpath {
```

When the pod is on a different node a HBONE HTTP Connect overlay pool is created between
both nodes.

```rust
    // outbound.rs
    let mut connection = self.pi.pool.connect(pool_key.clone(), connect).await?;

    let mut f = http_types::proxies::Forwarded::new();
    f.add_for(remote_addr.to_string());

    let request = hyper::Request::builder()
        .uri(&req.destination.to_string())
        .method(hyper::Method::CONNECT)
        .version(hyper::Version::HTTP_2)
        .header(
            BAGGAGE_HEADER,
            baggage(&req, self.pi.cfg.cluster_id.clone()),
        )
        .header(FORWARDED, f.value().unwrap())
        .header(TRACEPARENT_HEADER, self.id.header())
        .body(Empty::<Bytes>::new())
        .unwrap();

    let response = connection.send_request(request).await?;

    let code = response.status();
    if code != 200 {
        return Err(Error::HttpStatus(code));
    }
    let mut upgraded = hyper::upgrade::on(response).await?;

    super::copy_hbone(
        &mut upgraded,
        &mut stream,
        &self.pi.metrics,
        transferred_bytes,
    )
    .instrument(trace_span!("hbone client"))
    .await
```

For inbound (15008) the request is handled via `serve_connect` on each new connection on this port, it filters the Method:CONNECT (HBONE),
fetch the workload with XDS, and apply any RBAC on L4 in the connection, if everything is ok `handle_inbound` starts a stream with `freebind_connection`
and the actual destination, the stream/response is finally forwarded back through the tunnel and the caller.
 
```rust
   // inbound.rs : 195
    Hbone(req) => match hyper::upgrade::on(req).await {
        Ok(mut upgraded) => {
            if let Err(e) = super::copy_hbone(
                &mut upgraded,
                &mut stream,
                &metrics,
                transferred_bytes,
            )
            .instrument(trace_span!("hbone server"))
            .await
            {
                error!(dur=?start.elapsed(), "hbone server copy: {}", e);
            }
        }
        Err(e) => {
            // Not sure if this can even happen
            error!(dur=?start.elapsed(), "No upgrade {e}");
        }
    },

```

### xDS configuration

Envoy discovery services and their corresponding APIs are referred to as xDS. Every configuration resource in the xDS API has a type associated with it.
This information is transported over the xDS transport API, but uses a custom ambient-specific type.

Can be fetched as:

```shell
$ kubectl -n istio-system exec -it ztunnel-nnd8m -- curl http://localhost:15000/config_dump | jq .workloads
```

`ambient-worker` and `appa`

```json
{
  "10.244.1.8": {
    "workloadIp": "10.244.1.8",
    "waypointAddresses": [],
    "gatewayAddress": null,
    "protocol": "HBONE",
    "name": "appa-bf7c45dcb-lmtg2",
    "namespace": "default",
    "trustDomain": "cluster.local",
    "serviceAccount": "appa",
    "workloadName": "appa",
    "workloadType": "deployment",
    "canonicalName": "appa",
    "canonicalRevision": "v1",
    "node": "ambient-worker",
    "nativeHbone": false,
    "authorizationPolicies": [],
    "status": "Healthy",
    "clusterId": "Kubernetes"
  },
}
```

`ambient-worker2` and `appb` with an waypoint L7 gateway on xDS:

```json
{
  "10.244.2.4": {
    "workloadIp": "10.244.2.4",
    "waypointAddresses": [
      "10.244.1.10"
    ],
    "gatewayAddress": null,
    "protocol": "HBONE",
    "name": "appb-76b56f7cb4-qkjl7",
    "namespace": "default",
    "trustDomain": "cluster.local",
    "serviceAccount": "appb",
    "workloadName": "appb",
    "workloadType": "deployment",
    "canonicalName": "appb",
    "canonicalRevision": "v1",
    "node": "ambient-worker2",
    "nativeHbone": false,
    "authorizationPolicies": [],
    "status": "Healthy",
    "clusterId": "Kubernetes"
  },
}
```


### Logging and metrics

`ambient-worker` metrics scrapping from source `appa`:

```shell
root@ztunnel-hq9lr:/# curl http://localhost:15020/metrics
# HELP istio_tcp_connections_opened The total number of TCP connections opened.
# TYPE istio_tcp_connections_opened counter
istio_tcp_connections_opened_total{
    reporter="source",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",
    destination_service="unknown",destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",
    destination_canonical_revision="v1",destination_workload_namespace="default",
    destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",
    response_flags="-",connection_security_policy="mutual_tls"} 29

istio_tcp_connections_opened_total{
    reporter="source",source_workload="gateway-istio",source_canonical_service="gateway-istio",source_canonical_revision="latest",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/gateway-istio",source_app="gateway-istio",source_version="latest",source_cluster="Kubernetes",
    destination_service="unknown",destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="istiod",destination_canonical_service="istiod",
    destination_canonical_revision="latest",destination_workload_namespace="istio-system",destination_principal="spiffe://cluster.local/ns/istio-system/sa/istiod",
    destination_app="istiod",destination_version="latest",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="unknown"} 2

# HELP istio_tcp_connections_closed The total number of TCP connections closed.
# TYPE istio_tcp_connections_closed counter
istio_tcp_connections_closed_total{
    reporter="source",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",
    destination_service="unknown",destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",
    destination_canonical_service="appb",destination_canonical_revision="v1",destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",
    destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 29

# HELP istio_tcp_received_bytes The size of total bytes received during request in case of a TCP connection.
# TYPE istio_tcp_received_bytes counter
istio_tcp_received_bytes_total{
    reporter="source",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",destination_service="unknown",
    destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",
    destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",destination_version="v1",
    destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 2320

# HELP istio_tcp_sent_bytes The size of total bytes sent during response in case of a TCP connection.
# TYPE istio_tcp_sent_bytes counter
istio_tcp_sent_bytes_total{
    reporter="source",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",source_principal="spiffe://cluster.local/ns/default/sa/appa",
    source_app="appa",source_version="v1",source_cluster="Kubernetes",destination_service="unknown",destination_service_namespace="unknown",
    destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",
    destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",
    destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 13832
```

Executing curl `http://appb:8000` passing traffic through the waypoint 10.244.1.10:

```shell
INFO outbound{id=c03cd45569f64624051f3ce1229fe0e4}: ztunnel::proxy::outbound: proxy to 10.96.101.203:8000 using HBONE via 10.244.1.10:15008 type ToServerWaypoint
INFO outbound{id=c03cd45569f64624051f3ce1229fe0e4}: ztunnel::proxy::outbound: complete dur=52.729529ms
INFO outbound{id=d6dfd9221fcaf31e48f21f3c8ae63dcb}: ztunnel::proxy::outbound: proxy to 10.96.101.203:8000 using HBONE via 10.244.1.10:15008 type ToServerWaypoint
INFO outbound{id=d6dfd9221fcaf31e48f21f3c8ae63dcb}: ztunnel::proxy::outbound: complete dur=4.564153ms
INFO outbound{id=9cee33aa037814fe3e741117bb8f7128}: ztunnel::proxy::outbound: proxy to 10.96.101.203:8000 using HBONE via 10.244.1.10:15008 type ToServerWaypoint
INFO outbound{id=9cee33aa037814fe3e741117bb8f7128}: ztunnel::proxy::outbound: complete dur=48.65811ms
INFO outbound{id=4bc65655c588c8ae46e122c754963dfc}: ztunnel::proxy::outbound: proxy to 10.96.101.203:8000 using HBONE via 10.244.1.10:15008 type ToServerWaypoint
INFO outbound{id=4bc65655c588c8ae46e122c754963dfc}: ztunnel::proxy::outbound: complete dur=52.020461ms
INFO outbound{id=76faadf5ecc091584f30e6d8c9cef142}: ztunnel::proxy::outbound: proxy to 10.96.101.203:8000 using HBONE via 10.244.1.10:15008 type ToServerWaypoint
INFO outbound{id=76faadf5ecc091584f30e6d8c9cef142}: ztunnel::proxy::outbound: complete dur=3.984971ms
INFO outbound{id=94b826c34ce67957114b48cd54eb231d}: ztunnel::proxy::outbound: proxy to 10.96.101.203:8000 using HBONE via 10.244.1.10:15008 type ToServerWaypoint
INFO outbound{id=94b826c34ce67957114b48cd54eb231d}: ztunnel::proxy::outbound: complete dur=46.497344ms
````

`ambient-worker-2` metrics receiving the requeset on `appb`:

```shell
# HELP istio_tcp_connections_opened The total number of TCP connections opened.
# TYPE istio_tcp_connections_opened counter
istio_tcp_connections_opened_total{
    reporter="destination",source_workload="appb-istio-waypoint",source_canonical_service="appb-istio-waypoint",source_canonical_revision="latest",
    source_workload_namespace="default",source_principal="spiffe://cluster.local/ns/default/sa/appb-istio-waypoint",source_app="appb-istio-waypoint",
    source_version="latest",source_cluster="Kubernetes",destination_service="unknown",destination_service_namespace="unknown",destination_service_name="unknown",
    destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",destination_workload_namespace="default",
    destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",
    request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 79
istio_tcp_connections_opened_total{
    reporter="destination",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",destination_service="unknown",
    destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",
    destination_canonical_revision="v1",destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",
    destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 4

# HELP istio_tcp_connections_closed The total number of TCP connections closed.
# TYPE istio_tcp_connections_closed counter
istio_tcp_connections_closed_total{
    reporter="destination",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",destination_service="unknown",
    destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",
    destination_canonical_revision="v1",destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",
    destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 4
istio_tcp_connections_closed_total{
    reporter="destination",source_workload="appb-istio-waypoint",source_canonical_service="appb-istio-waypoint",source_canonical_revision="latest",
    source_workload_namespace="default",source_principal="spiffe://cluster.local/ns/default/sa/appb-istio-waypoint",source_app="appb-istio-waypoint",
    source_version="latest",source_cluster="Kubernetes",destination_service="unknown",destination_service_namespace="unknown",
    destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",
    destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",
    destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 78

# HELP istio_tcp_received_bytes The size of total bytes received during request in case of a TCP connection.
# TYPE istio_tcp_received_bytes counter
istio_tcp_received_bytes_total{
    reporter="destination",source_workload="appb-istio-waypoint",source_canonical_service="appb-istio-waypoint",source_canonical_revision="latest",
    source_workload_namespace="default",source_principal="spiffe://cluster.local/ns/default/sa/appb-istio-waypoint",source_app="appb-istio-waypoint",
    source_version="latest",source_cluster="Kubernetes",destination_service="unknown",destination_service_namespace="unknown",
    destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",
    destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",destination_version="v1",
    destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 36179
istio_tcp_received_bytes_total{
    reporter="destination",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",destination_service="unknown",
    destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",
    destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",destination_version="v1",
    destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 320

# HELP istio_tcp_sent_bytes The size of total bytes sent during response in case of a TCP connection.
# TYPE istio_tcp_sent_bytes counter
istio_tcp_sent_bytes_total{
    reporter="destination",source_workload="appa",source_canonical_service="appa",source_canonical_revision="v1",source_workload_namespace="default",
    source_principal="spiffe://cluster.local/ns/default/sa/appa",source_app="appa",source_version="v1",source_cluster="Kubernetes",destination_service="unknown",
    destination_service_namespace="unknown",destination_service_name="unknown",destination_workload="appb",destination_canonical_service="appb",
    destination_canonical_revision="v1",destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/appb",
    destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 1332
istio_tcp_sent_bytes_total{
    reporter="destination",source_workload="appb-istio-waypoint",source_canonical_service="appb-istio-waypoint",source_canonical_revision="latest",
    source_workload_namespace="default",source_principal="spiffe://cluster.local/ns/default/sa/appb-istio-waypoint",source_app="appb-istio-waypoint",
    source_version="latest",source_cluster="Kubernetes",destination_service="unknown",destination_service_namespace="unknown",destination_service_name="unknown",
    destination_workload="appb",destination_canonical_service="appb",destination_canonical_revision="v1",destination_workload_namespace="default",
    destination_principal="spiffe://cluster.local/ns/default/sa/appb",destination_app="appb",destination_version="v1",destination_cluster="Kubernetes",
    request_protocol="tcp",response_flags="-",connection_security_policy="mutual_tls"} 65208
```

Receiving from waypoint and forward to the pod IP:port

```shell
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
INFO inbound{peer_ip=10.244.1.10 peer_id=spiffe://cluster.local/ns/default/sa/appb-istio-waypoint}: ztunnel::proxy::inbound: got CONNECT request to 10.244.2.4:80
```


