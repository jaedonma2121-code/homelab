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
     |          |
     |          +----> Open WebUI
     |                     |
     |                     v
     |                   Ollama
     |                     |
     |                     v
     |                NVIDIA GPU
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
  +----------------------+------------------+
  |                      |                  |
  v                      v                  v
File Browser          Open WebUI        Other web apps
                          |
                          | internal API
                          v
                       Ollama
                          |
                          v
                      GPU runtime
~~~

Only the reverse-proxy layer is intended to be the normal web ingress point.

## AI Inference Architecture

The AI workload uses a dedicated virtual machine with GPU passthrough.

~~~text
Proxmox VE
    |
    | PCIe passthrough
    v
AI virtual machine
    |
    +----> Open WebUI
    |          |
    |          v
    |       Ollama
    |          |
    |          v
    |      NVIDIA GPU
    |
    +----> persistent application data
~~~

The design separates:

| Layer | Responsibility |
|---|---|
| Reverse proxy | HTTPS ingress and request routing |
| Open WebUI | User interface, sessions, and application workflows |
| Ollama | Model serving and inference API |
| NVIDIA GPU | Hardware acceleration |
| Proxmox | Virtualisation and device passthrough |

This avoids coupling the browser-facing application to the GPU runtime and provides a clear boundary for troubleshooting.

The repository does not publish the VM identifier, GPU PCI identifiers, internal addresses, model inventory, or live Docker configuration.

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

## TLS and Remote Access

The current architecture separates network access from web TLS.

~~~text
Approved remote client
       |
       | authenticated overlay network
       v
Nginx reverse proxy
       |
       | HTTPS / TLS termination
       v
Internal application
~~~

The reverse proxy uses a publicly trusted certificate so standard clients do not need to install a lab-specific root CA. Certificate issuance is performed through DNS validation.

Private overlay DNS resolves service names to overlay-network addresses for approved clients. Public DNS is not used to publish those private addresses.

Certificate private keys, DNS credentials, live hostnames, and overlay addresses are intentionally excluded from this repository.

## Remote Access

Remote access requires an authenticated private network. The overlay network provides connectivity to the reverse proxy without publishing the private service network or application management interfaces.

~~~text
Remote device
     |
     | authenticated overlay network
     v
Nginx
     |
     v
Internal service
~~~

Direct public exposure of backend application ports is intentionally outside this architecture.

## Failure Domains

| Layer | Example failure |
|---|---|
| Physical/network | Upstream connectivity unavailable |
| Proxmox | Bridge, routing, or passthrough configuration failure |
| NAT | Incorrect translation |
| Forwarding | Packet filtering/forwarding failure |
| Nginx | Listener or virtual-host configuration failure |
| TLS | Certificate, trust, or SNI mismatch |
| Application | Open WebUI or File Browser unavailable |
| Inference | Ollama unavailable or model load failure |
| GPU | Passthrough, driver, or runtime failure |
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
- authenticated overlay networking
- private DNS for overlay clients
- GPU workload monitoring
- resource quotas for AI workloads

## Architecture Principle

> Expose capabilities, not infrastructure.

The public repository should explain the engineering decisions while withholding information required to enumerate or target the live home network.
