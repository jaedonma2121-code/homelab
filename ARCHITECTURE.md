# Architecture

This document describes the homelab using sanitised logical names.

It deliberately omits real IP addresses, hostnames, interface names, MAC addresses, physical disk identifiers, container IDs, exact port-forwarding mappings, certificate identities, and credentials.

## High-Level Topology

~~~text
Upstream LAN
     |
     v
Proxmox VE
     |
     | private bridge
     v
Private service network
     |
     +----> Nginx reverse proxy
     |          |
     |          +----> File Browser Quantum
     |
     +----> Storage services
                |
                +----> host-mounted data
~~~

## Network Segmentation

LAN-facing zone: <LAN_SUBNET>

Private service zone: <SERVICE_SUBNET>

The real values are not published.

## Proxmox as the Layer-3 Boundary

The Proxmox host performs IPv4 forwarding between the LAN-facing interface and private service bridge.

~~~text
LAN
 |
 v
Proxmox routing table
 |
 v
Private service network
~~~

## Source NAT

Private services can use controlled outbound access through the upstream network.

~~~text
Private service
      |
      v
Proxmox
      |
      | source NAT / masquerading
      v
Upstream LAN
~~~

The live NAT commands and subnets are excluded.

## Destination NAT

Inbound traffic is selectively forwarded from the LAN-facing boundary to approved services.

~~~text
Client
  |
  | approved ingress
  v
Proxmox
  |
  | DNAT
  v
Approved internal service
~~~

Exact live mappings are intentionally withheld.

## Web Application Ingress

~~~text
Client
  |
  | HTTPS
  v
Proxmox ingress
  |
  | controlled forwarding
  v
Nginx
  |
  | TLS termination
  | HTTP reverse proxy
  v
Internal application
~~~

Only the reverse-proxy layer is intended to be the normal web ingress point.

## File Browser Quantum

File Browser Quantum runs inside a private LXC.

~~~text
Nginx
  |
  | internal HTTP
  v
File Browser Quantum
  |
  v
Service storage mount
~~~

The service is managed by systemd.

## Storage Architecture

The physical data disk is mounted by the Proxmox host. A directory from that host mount is presented to the application container through an LXC mount point.

~~~text
Physical data volume
        |
        v
Proxmox host mount
        |
        | LXC mount point
        v
Application container
        |
        +--> File Browser
        |
        +--> Samba-compatible access
~~~

This avoids giving the application container direct ownership of the physical block device.

## Private PKI and TLS

~~~text
Private Root CA
       |
       v
Internal server certificate
       |
       +--> Nginx HTTPS
       |
       +--> Internal service hostnames
~~~

The CA private key remains secured and is never committed to Git.

Client devices that use the private services must trust the CA certificate. This includes computers and mobile devices.

## Remote Access

Internal private service naming is intended for the home network.

~~~text
Remote device
     |
     | authenticated VPN / private network
     v
Home network
     |
     v
Nginx
     |
     v
Internal service
~~~

Direct public exposure of the file-management application is intentionally outside this architecture.

## Failure Domains

| Layer | Example failure |
|---|---|
| Physical/network | Upstream connectivity unavailable |
| Proxmox | Bridge or routing configuration failure |
| NAT | Incorrect translation |
| Forwarding | Packet filtering/forwarding failure |
| Nginx | Listener or virtual-host configuration failure |
| TLS | Certificate, trust, or SNI mismatch |
| Application | File Browser unavailable |
| Storage | Host mount unavailable |
| Authentication | Invalid credentials or policy |

## Hardening Roadmap

- default-deny forwarding policy
- stateful connection tracking
- explicit source allowlists
- management-plane isolation
- VLAN segmentation
- dedicated physical network uplink
- firewall logging
- centralised monitoring
- backup verification
- secrets management
- automated certificate lifecycle
- VPN-based remote access

## Architecture Principle

> Expose capabilities, not infrastructure.

The public repository should explain the engineering decisions while withholding information required to enumerate or target the live home network.
