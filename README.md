# Home Lab Infrastructure

> Proxmox VE / Linux Networking / Reverse Proxy / TLS / AI Inference / Storage / Infrastructure Troubleshooting

A security-conscious home lab demonstrating practical infrastructure engineering with Proxmox VE, Linux routing and NAT, isolated service networking, Nginx, self-hosted AI inference, File Browser Quantum, Samba-compatible storage, authenticated overlay networking, publicly trusted TLS, and systemd-managed services.

This repository is intentionally sanitised for public viewing. It documents architecture, engineering decisions, troubleshooting methodology, and operational patterns without publishing live IP addresses, hostnames, usernames, filesystem identifiers, certificate material, or port mappings from the real environment.

## Security-first documentation policy

Never commit real LAN or private-subnet addresses, real DNS names, public IPs, MAC addresses, Wi-Fi credentials, passwords, tokens, private keys, live certificates, physical disk identifiers, exact DNAT/port mappings, raw production configuration, databases, backups, or unredacted logs.

Examples use symbolic values such as <LAN_SUBNET>, <SERVICE_SUBNET>, <HOST_ADDRESS>, <SERVICE_ADDRESS>, and <AI_ENDPOINT>.

## Architecture Overview

~~~text
Home LAN
   |
   | controlled connectivity
   v
Proxmox VE
   |
   | private service network
   +----> Nginx reverse proxy
   |           |
   |           +----> File Browser Quantum
   |           |
   |           +----> Open WebUI
   |                         |
   |                         v
   |                      Ollama
   |                         |
   |                         v
   |                    NVIDIA GPU
   |
   +----> storage services
   |           |
   |           +----> host-mounted data
   |
   +----> Minecraft Java server
              ^
              |
       authenticated overlay
       + controlled routing
~~~

The architecture separates upstream connectivity, host-level routing/NAT, private services, application ingress, AI inference, and persistent storage.

## Core Components

| Component | Responsibility |
|---|---|
| Proxmox VE | Hypervisor and Layer-3 network boundary |
| Linux bridge | Private service-network connectivity |
| iptables | NAT, DNAT, and packet forwarding |
| Nginx | HTTP reverse proxy and TLS termination |
| Open WebUI | Web interface for self-hosted AI inference |
| Ollama | Local LLM inference runtime |
| NVIDIA GPU passthrough | Dedicated hardware acceleration for AI workloads |
| File Browser Quantum | Web-based file management |
| Samba | Network file-sharing interface |
| systemd | Persistent service management |
| Authenticated overlay network | Private remote connectivity for approved clients |
| Network routing layer | Controlled forwarding to approved non-HTTP services |
| Minecraft Java Server | Dedicated game-server workload |
| tmux | Persistent interactive Minecraft console |
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
  +--------------------+-------------------+
  |                    |                   |
  v                    v                   v
File Browser        Open WebUI        Other services
                         |
                         | internal HTTP
                         v
                      Ollama
                         |
                         v
                     GPU runtime
~~~

The reverse proxy owns client-facing TLS while application backends remain on the private service network.

## Self-Hosted AI Service

The lab includes a dedicated AI workload for local LLM inference.

~~~text
Approved client
      |
      | authenticated overlay
      v
Nginx reverse proxy
      |
      | HTTPS / TLS termination
      v
Open WebUI
      |
      | internal API
      v
Ollama
      |
      | GPU acceleration
      v
NVIDIA GPU
~~~

The AI service separates the user-facing application from the inference runtime. Open WebUI provides the browser interface and session/authentication layer, while Ollama manages model execution. GPU passthrough allows the virtualised workload to use dedicated NVIDIA hardware without exposing the GPU directly to other services.

The public repository documents the workload pattern, not the live endpoint, VM identifier, GPU identifiers, model inventory, or network mappings.

### AI engineering highlights

- Virtualised AI workload on Proxmox
- PCIe GPU passthrough
- NVIDIA GPU acceleration
- Ollama inference runtime
- Open WebUI application layer
- Reverse-proxy ingress
- TLS termination
- Private backend networking
- Separation of application and inference responsibilities
- Persistent application data through Docker volumes

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

## Minecraft Server

The lab includes a dedicated Minecraft Java Edition service on the private service network. Remote clients reach it through the authenticated overlay and controlled routing layer rather than through public router port-forwarding.

~~~text
Approved remote client
        |
        | authenticated overlay
        v
Existing routing peer
        |
        | controlled forwarding
        v
Minecraft Java server
        |
        v
tmux-backed service console
~~~

Minecraft is intentionally kept outside the HTTP reverse-proxy path because the game server uses its own protocol rather than HTTP/HTTPS. The server is managed by systemd and uses tmux to provide a persistent interactive console without requiring RCON or a web administration panel.

The public repository omits live addresses, hostnames, container IDs, routing entries, overlay addresses, client identifiers, and network policy details.

See [services/minecraft/README.md](services/minecraft/README.md) for the sanitised service architecture.

## Service Management

Application services are managed with systemd or Docker depending on workload requirements. Long-running services define explicit startup behaviour, restart policy, configuration, and persistent storage.

## Remote Client Access

Approved computers and mobile devices reach the reverse proxy through an authenticated private-access overlay.

The repository documents the access pattern rather than the live implementation details. Private DNS records, overlay addresses, device identifiers, and credentials are deliberately omitted.

~~~text
Approved client
      |
      | authenticated overlay
      v
Nginx reverse proxy
      |
      | HTTPS
      v
Internal service
~~~

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
- PCIe GPU passthrough
- NVIDIA GPU acceleration
- Local LLM inference
- Ollama
- Open WebUI
- Docker containerisation
- LXC storage mount design
- Shared storage architecture
- systemd service management
- Linux troubleshooting
- Root-cause analysis
- Service isolation
- Dedicated Minecraft Java hosting
- tmux-based interactive service administration
- Security-conscious public documentation

## Documentation

| Document | Purpose |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Sanitised architecture and traffic-flow design |
| [RUNBOOKS/TROUBLESHOOTING.md](RUNBOOKS/TROUBLESHOOTING.md) | Troubleshooting cases and RCA methodology |
| [SECURITY.md](SECURITY.md) | Public-repository security and redaction policy |
| [services/ai/README.md](services/ai/README.md) | Sanitised AI service architecture and deployment pattern |
| [docs/decisions/ADR-004-ai-inference.md](docs/decisions/ADR-004-ai-inference.md) | Decision record for virtualised GPU-backed AI inference |

## Public Portfolio Principle

The repository documents how the infrastructure was designed and engineered, not a blueprint of the live home network. A recruiter can see the relevant technical skills without receiving the real network layout, service addresses, certificate identities, or forwarding rules.

## Remote Access and Public TLS

The lab separates remote connectivity from web application ingress:

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
