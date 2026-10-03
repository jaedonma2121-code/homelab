# Home Lab Infrastructure

> Proxmox VE / Linux Networking / Reverse Proxy / TLS / Storage / Infrastructure Troubleshooting

A security-conscious home lab demonstrating practical infrastructure engineering with Proxmox VE, Linux routing and NAT, isolated service networking, Nginx, File Browser Quantum, Samba-compatible storage, authenticated overlay networking, publicly trusted TLS, and systemd-managed services.

This repository is intentionally sanitised for public viewing. It documents architecture, engineering decisions, troubleshooting methodology, and operational patterns without publishing live IP addresses, hostnames, usernames, filesystem identifiers, certificate material, or port mappings from the real environment.

## Security-first documentation policy

Never commit real LAN or private-subnet addresses, real DNS names, public IPs, MAC addresses, Wi-Fi credentials, passwords, tokens, private keys, live certificates, physical disk identifiers, exact DNAT/port mappings, raw production configuration, databases, backups, or unredacted logs.

Examples use symbolic values such as <LAN_SUBNET>, <SERVICE_SUBNET>, <HOST_ADDRESS>, and <SERVICE_ADDRESS>.

## Architecture Overview

~~~text
Home LAN
   |
   | HTTPS
   v
Proxmox VE
   |
   | private service network
   +----> Nginx reverse proxy
   |           |
   |           +----> File Browser Quantum
   |
   +----> storage services
              |
              +----> host-mounted data
~~~

The architecture separates upstream connectivity, host-level routing/NAT, private services, application ingress, and persistent storage.

## Core Components

| Component | Responsibility |
|---|---|
| Proxmox VE | Hypervisor and Layer-3 network boundary |
| Linux bridge | Private service-network connectivity |
| iptables | NAT, DNAT, and packet forwarding |
| Nginx | HTTP reverse proxy and TLS termination |
| File Browser Quantum | Web-based file management |
| Samba | Network file-sharing interface |
| systemd | Persistent service management |
| Authenticated overlay network | Private remote connectivity for approved clients |
| Publicly trusted TLS | Client-trusted HTTPS at the reverse-proxy boundary |
| Host-mounted storage | Shared data volume presented to services |

## Network Design

The lab uses two logical network zones.

LAN-facing network: <LAN_SUBNET>

Private service network: <SERVICE_SUBNET>

The real values are intentionally omitted.

## Routing and NAT

The Proxmox host provides the routing boundary between the upstream LAN and private service network.

Conceptually:

~~~text
LAN client
   |
   v
Proxmox routing stack
   |
   +--> selective DNAT --> reverse proxy
   |
   +--> selective DNAT --> approved service
   |
   +--> private-network egress --> upstream NAT
~~~

The live DNAT rules are deliberately not committed.

## Nginx Reverse Proxy and TLS

Web applications are exposed through Nginx rather than publishing every application port directly.

~~~text
Client
  |
  | HTTPS
  v
Nginx
  |
  | TLS termination
  | HTTP reverse proxy
  v
Internal application
~~~

The current web ingress uses a publicly trusted certificate at Nginx, while private remote access is provided through an authenticated overlay network. Earlier internal-only TLS used a private CA; its private key and certificate material remain outside the repository.

## File Browser Quantum

File Browser Quantum is deployed as an isolated service on the private service network. It runs as a systemd-managed service, uses an internal HTTP listener, is reached through Nginx for HTTPS access, and uses the existing host-mounted data volume.

~~~text
Client
  |
  | HTTPS
  v
Nginx
  |
  | HTTP
  v
File Browser Quantum
  |
  v
LXC storage mount
  |
  v
Host-mounted data volume
~~~

The real mount path, device name, container identifier, application address, and application port are intentionally omitted.

### Storage design

The physical data volume is mounted and managed by the Proxmox host. Services receive the host-mounted directory through an LXC mount point.

~~~text
Physical data volume
        |
        v
Proxmox host filesystem mount
        |
        | LXC mount point
        v
Service container
        |
        v
File Browser / Samba
~~~

If multiple services access the same filesystem, they should use the correctly mounted filesystem path rather than independently mounting the physical device.

## Service Management

Application services are managed with systemd rather than manually started from an interactive shell. A working directory, configuration path, restart policy, and boot-time startup are defined explicitly.

## Family and Client Access

Internal HTTPS using a private CA requires each client device to trust the CA. This applies to computers and phones.

The CA certificate may be installed on trusted client devices. The CA private key must never be installed on client devices.

For access outside the home network, use an authenticated private-access layer. The repository documents the pattern only; live overlay addresses, DNS records, and credentials are intentionally omitted.

## Engineering Highlights

- Proxmox VE virtualisation
- Layer-3 network segmentation
- Linux bridge networking
- IPv4 forwarding
- Source NAT / masquerading
- Destination NAT
- Packet forwarding
- Nginx reverse proxy
- TLS/SSL termination
- Authenticated overlay networking
- Split-DNS service discovery
- DNS-validated public TLS
- LXC storage mount design
- Shared storage architecture
- systemd service management
- Linux troubleshooting
- Root-cause analysis
- Service isolation
- Security-conscious public documentation

## Documentation

| Document | Purpose |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Sanitised architecture and traffic-flow design |
| [RUNBOOKS/TROUBLESHOOTING.md](RUNBOOKS/TROUBLESHOOTING.md) | Troubleshooting cases and RCA methodology |
| [SECURITY.md](SECURITY.md) | Public-repository security and redaction policy |

## Public Portfolio Principle

The repository documents how the infrastructure was designed and engineered, not a blueprint of the live home network. A recruiter can see the relevant technical skills without receiving the real network layout, service addresses, certificate identities, or forwarding rules.

## Remote Access and Public TLS

The lab now separates remote connectivity from web application ingress:

~~~text
Approved client
      |
      | authenticated overlay network
      v
Nginx reverse proxy
      |
      | HTTPS / TLS termination
      v
Internal application
~~~

Service hostnames are resolved privately for approved overlay-network clients. Public DNS is not used to publish the private overlay addresses. Certificate issuance uses DNS validation, so no DNS credential or certificate private key belongs in this repository.

The live domain, overlay addresses, DNS credentials, certificate identities, and Nginx virtual-host configuration are intentionally excluded.
