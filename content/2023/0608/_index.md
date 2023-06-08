+++
title = "Secure gRPC services with mTLS"
description = "Hacking golang gRPC services and mTLS setup"
date = "2023-06-08"
+++

## Introduction

Authentication is part of the AAA (Authn, Authz, Audit) security framework, and a basic functionality for each
multi tenant modern system, the usage of gRPC as a channel of communication for east-west trafic allow the developer
to enable built-in mechanisms capable to not only secure the traffic via TLS encryption but to enable authentication
using SSL/TLS X.509 certificates.

This post will detail and demonstrate a client/server implementation that uses gRPC as a protocol of communication
and uses these TLS certificates for channels credentials.

The last part introduces `cert-manager` another CNCF project for PKI management inside Kubernetes clusters,
and its going to move the application to containers and use automatic generated certs instead.

## TLS and SSL

The goal of PKI (Public Key Infrastructure) is the management of public keys and digital certificates 
on a specific company infrastructure.


The most well known client tool for this task is `openssl`, these SSL/TLS toolkits are very used 
in the CLI for management of keys and certificates, on this post it will be used [Cloudfare's PKI and TLS toolkit](https://github.com/cloudflare/cfssl).

### Generating certificates

## gRPC

### Mutual TLS gRPC

