# ADR-003: Private Remote Access with Publicly Trusted TLS

- **Status:** Accepted
- **Decision:** Use an authenticated overlay network for remote access and a publicly trusted certificate for service HTTPS.
- **Scope:** Remote access and web ingress

## Context

The lab needs remote access from trusted personal devices without publishing private service addresses or exposing management interfaces directly to the public internet.

A private CA works well for a closed LAN, but it adds a certificate-trust requirement to every client device. Remote access also needs a separate authenticated network boundary.

## Decision

Use two separate concerns:

1. **Overlay network** — provides authenticated private connectivity between approved devices and the reverse-proxy host.
2. **Publicly trusted TLS** — provides HTTPS for the service hostnames, with certificate issuance performed through DNS validation.

Conceptually:

```text
Remote client
     |
     | authenticated overlay
     v
Reverse proxy
     |
     | HTTPS / TLS termination
     v
Internal application
```

Public DNS remains responsible for the public domain namespace. The overlay network provides private name resolution for service records that should resolve to the overlay address rather than a public address.

## Security Properties

- No requirement to expose application management ports to the public internet.
- Service applications remain on the private service network.
- Remote access is authenticated at the overlay-network layer.
- HTTPS uses a certificate trusted by standard clients.
- DNS validation avoids placing certificate private keys or DNS credentials in the repository.

## Consequences

### Positive

- Remote access works without publishing the private service subnet.
- Phones and computers can use normal HTTPS trust without installing the lab CA.
- Nginx remains the single web-ingress point.
- Public DNS and private overlay DNS can serve different purposes.

### Negative

- Remote access depends on the overlay-network control plane.
- Certificate renewal requires reliable DNS validation automation.
- Split DNS must be maintained consistently for approved clients.
- The reverse proxy remains a critical dependency.

## Public Documentation Boundary

The repository intentionally does **not** publish:

- the real domain name
- public or private IP addresses
- overlay IP addresses
- DNS tokens
- setup keys
- certificate private keys
- live Nginx configuration
- exact forwarding rules
- personal device identifiers

Only the architecture and operational pattern are documented.
