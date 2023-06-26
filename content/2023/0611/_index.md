+++
title = "SPIFFE/SPIRE for gRPC authz"
description = "Strong authn and authz with go-spiffe"
date = "2023-06-11"
+++

## Introduction

The Zero trust security model has a motto: "never trust, always verify". As notices in the last post
verification of x509 in a gRPC scenario without much libraries are dangerous, the lack of a good
identification of the service can be a problem for authorization and authentication of the party
trying to access a particular service. On this post lets continue the testing and validation
of a few cloud native security technologies offered for Golang related to service identification.

## SPIFFE (Secure Production Identity Framework For Everyone)

[SPIFFE](https://spiffe.io) and SPIRE aim to strengthen the identification of software components
ina a common way that can be leveraged across distributed systems by anyone, anywhere.

Environment growth and services dynamics can help to make the service identification a challenge,
policy enforcement and lifecycle management of the PKI, services IDs and other traits become hard.

The key for this is a trustworthy identity that can in a solid and identifiable way represent a workload
that claims for this identity. In this framework this identity is called SPIFFE ID, and as the format
`spiffe://<domain>/<workload>`, other important part is the SPIFFE verifiable identity document (SVID).
it supports both X509 and JWT, for the first the SPIFFEID will exist as a URI in the Subject Alternative
Name (SAN) extension.


### Architecture

{{< mermaid >}}
flowchart LR
    agent_1-->Server_API
    agent_2-->Server_API

    subgraph agent_1[Agent]
      workloadapi1
    end

    workload1[Workload] --> workloadapi1[API]

    subgraph agent_2[Agent]
      workloadapi2
    end

    workload2[Workload] --> workloadapi2[API]

        
    classDef ctrl fill:#1568ca,color:white,stroke-width:1px,stroke:#dbffff;
    classDef normal fill:#007cff,color:white,stroke-width:1px,stroke:#dbfff;
    classDef cluster fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;

    class agent_1,agent_2 normal;
    class Server_API normal;
    class workload1,workload2 normal;
    class workloadapi1,workloadapi2 ctrl;

{{< /mermaid >}}
    
Server API manages and issues identities in a SPIFFE trust domain, hold information
about its agents and workloads. Uses registration entites for Agents and Workloads

Workload API, provides the following

* Its identity, described as a SPIFFE ID.
* A private key tied to that ID that can be used to sign data on behalf of the workload. 
* A corresponding short-lived X.509 certificate is also created, the X509-SVID. This can be used to establish TLS or otherwise authenticate to other workloads.
* A set of certificates – known as a trust bundle – that a workload can use to verify an X.509-SVID presented by another workload

Agents runs on each node, they request SVID from the server and cache until workload requests its SVID,
exposing the SPIFFE workload API to workloads on a node and attests the identity of the workload. And
finally they provide identified workload with their SVIDs.

In the official docs the following [sequece](https://spiffe.io/docs/latest/spire-about/spire-concepts/) exists for the lifecycle of the SVID:

{{< mermaid >}}
sequenceDiagram
    participant Workload
    participant Agent
    participant Server 
    participant Cloud

    Agent->>Server: Performs node attestation (via TLS)
    Server->>Cloud: Node resolution to verify additional properties
    Cloud-->>Server: ACK
    Server->>Agent: Issues an SVID. 
    Agent-->>Server: Obtain the registration entries it is authorized for.
    Agent->>Agent: Completes mTLS handshake and auth the server with bundle
    Agent->>Server: Agent sends the workload CSR
    activate Server
    Server-->>Agent: Returns the workload SVID to the client
    deactivate Server
    Workload->>Agent: Request a SVID
    activate Agent
    Agent-->>Workload: Returns the SVID (JWT/X509)
    deactivate Agent
{{< /mermaid >}}

## go-spiffe

All the code exists on [Tutorial SPIFEE](https://github.com/knabben/tutorial-istio-sec/tree/main/2-spiffe). If
you are interested in installing Kind and giving a try in the specs the requirement is installing the
[the server](https://github.com/spiffe/spire-tutorials/tree/main/k8s/quickstart), agent pods first.

For this example it was created 2 ServiceAccounts, one for the `client` and other for the `server`, this
will allow us to create a different SPIFFE ID for each in the format `spiffe://opssec.in/ns/spire/sa/default/client||server`

The codebase for the client is similar to the normal mTLS, the difference here is the transport credentials
now receive a ACL for Server SPIFFE ID authorization, the `allowID` must have the correct SPIFFEID
otherwise some odd TLS error like `tls: bad certificate` can happen.

```golang
source, err := workloadapi.NewX509Source(ctx, workloadapi.WithClientOptions(workloadapi.WithAddr(*sockPath)))
if err != nil {
return fmt.Errorf("unable to create X509Source: %w", err)
}
defer source.Close()

// Allowed SERVER SPIFFE ID
serverID := spiffeid.RequireFromString(*allowID)

// Dial the server with credentials that do mTLS and verify that presented certificate has SPIFFE ID
conn, err := grpc.DialContext(ctx, *host, grpc.WithTransportCredentials(
    grpccredentials.MTLSClientCredentials(source, source, tlsconfig.AuthorizeID(serverID)),
))
if err != nil {
    return fmt.Errorf("failed to dial: %w", err)
}
```
 
For the server the same mTLS authorization ACL is required, now with the `client` credentials. 

```golang
source, err := workloadapi.NewX509Source(ctx, workloadapi.WithClientOptions(workloadapi.WithAddr(*sockPath)))
if err != nil {
    return fmt.Errorf("unable to create X509Source: %w", err)
}
defer source.Close()

// Allowed CLIENT SPIFFE ID
clientID := spiffeid.RequireFromString(*allowID)

// Create a server with credentials that do mTLS and verify that the presented certificate has SPIFFE ID
s := grpc.NewServer(grpc.Creds(
    grpccredentials.MTLSServerCredentials(source, source, tlsconfig.AuthorizeID(clientID)),
))
```

The first workloadapi connection do the workload attestation using as a selector the serviceaccount created
for each service. For this we need to ensure a registration entry is created in the API server, for each
service account used and the node.

```shell
kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://opssec.in/ns/default/sa/client||server \
	-parentID spiffe://opssec.in/ns/spire/sa/spire-agent \
	-selector k8s:ns:default \
	-selector k8s:sa:client||server
```

### Conclusion

![opa](./images/opa.png?width=1024 "opa")

Things start to get more interesting when complex implementation and integration of the technologies happen, 
This is an example of the [Envoy+OPA](https://spiffe.io/docs/latest/microservices/envoy-opa/readme/) application
the tutorial shows how to configure Envoy SDS (Secret Discovery Service) for automatic retrieve of these secrets
TLS certificated, trusted CA etcs, the SPIRE agent can be configured as an SDS provider for Envoy, allowing it
to directly provide Envoy with the key material it needs to provide TLS auth. Having this layer decoupled
of the codebase is a good first steps on the separation of concerns and well defined boundaries when
developing microservices in the cloud.

