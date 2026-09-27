# ADR-001: Separate LAN and Private Service Networks

- **Status:** Accepted
- **Decision:** Use a dedicated private service subnet behind the Proxmox host.
- **Scope:** Home-lab networking

## Context

The lab hosts multiple services with different exposure requirements. Keeping every service directly on the LAN would make traffic boundaries and service exposure less explicit.

## Decision

Use two logical networks:

```text
LAN / Client Network
        |
        | Proxmox routing boundary
        v
Private Service Network
```

The Proxmox host provides:

- Layer-3 routing
- IPv4 forwarding
- NAT
- DNAT
- controlled service exposure

Nginx Proxy Manager provides the HTTP/TLS application boundary.

## Consequences

### Positive

- Clear trust boundaries
- Explicit service exposure
- Easier troubleshooting
- Centralised reverse-proxy policy
- Good foundation for future VLANs/firewall rules

### Negative

- The Proxmox host becomes a network dependency.
- Host-level firewall rules require careful lifecycle management.
- A future production design would benefit from dedicated firewall infrastructure.

## Future Evolution

Potential next steps:

- default-deny forwarding
- stateful firewall policy
- VLAN segmentation
- dedicated management network
- monitoring and centralised logging
