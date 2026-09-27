# Home Lab Infrastructure

> **Infrastructure-as-Code / Networking / Reverse Proxy / Linux Administration / Cybersecurity**

A production-inspired home lab built around **Proxmox VE**, Linux networking, Layer-3 subnet separation, `iptables` NAT, and **Nginx Proxy Manager**.

The environment separates a LAN-facing network from an isolated private service network, while exposing only explicitly mapped services through controlled host-level forwarding and reverse-proxy entry points.

## Architecture Overview

```
                              ┌─────────────────────────┐
                              │       LAN / Client Network       │
                              │      <LAN_CIDR>     │
                              └────────────┬────────────┘
                                           │
                                  <LAN_HOST_IP>/24
                                           │
                              ┌────────────▼────────────┐
                              │       Proxmox Host       │
                              │                          │
                              │  <LAN_INTERFACE>                  │
                              │  <LAN_HOST_IP>/24       │
                              │                          │
                              │  ┌────────────────────┐  │
                              │  │ iptables           │  │
                              │  │ NAT / DNAT /       │  │
                              │  │ Forwarding         │  │
                              │  └─────────┬──────────┘  │
                              │            │             │
                              │  vmbr0     │             │
                              │ <PRIVATE_GATEWAY_IP>/24            │
                              └────────────┬─────────────┘
                                           │
                    ───────────────────────┼──────────────────────
                                           │
                              Private Service Network
                                   <PRIVATE_SERVICE_CIDR>
                                           │
               ┌───────────────────────────┼────────────────────────┐
               │                           │                        │
       ┌───────▼────────┐         ┌────────▼────────┐      ┌────────▼────────┐
       │ Samba Storage  │         │  Filebrowser    │      │ Nginx Proxy     │
       │ <SAMBA_IP>   │         │ <FILEBROWSER_IP>    │      │ Manager         │
       │ TCP 139, 445   │         │ TCP 8080        │      │ <REVERSE_PROXY_IP>    │
       └────────────────┘         └─────────────────┘      │ 80 / 81 / 443   │
                                                           └─────────────────┘
```

## Technical Summary

| Component | Address | Role | Exposed Ports |
|---|---|---|---|
| Proxmox Host / Wi-Fi | `<LAN_HOST_IP>` | LAN gateway / NAT / forwarding | 80, 81, 139, 445, 9443 |
| Proxmox `vmbr0` | `<PRIVATE_GATEWAY_IP>` | Private service-network gateway | — |
| Samba | `<SAMBA_IP>` | Network storage | TCP 139, 445 |
| Filebrowser | `<FILEBROWSER_IP>` | Web file management | TCP 8080 |
| Nginx Proxy Manager | `<REVERSE_PROXY_IP>` | Reverse proxy / TLS termination | TCP 80, 81, 443 |

## Network Address Allocation

| Network | CIDR | Purpose |
|---|---|---|
| LAN | `<LAN_CIDR>` | Upstream Wi-Fi/LAN |
| Private | `<PRIVATE_SERVICE_CIDR>` | Internal services |

| Interface | Address | Function |
|---|---|---|
| `<LAN_INTERFACE>` | `<LAN_HOST_IP>/24` | LAN-facing interface |
| `vmbr0` | `<PRIVATE_GATEWAY_IP>/24` | Private service bridge |

## Port Allocation

| Public Endpoint | Protocol | Destination | Service |
|---|---|---|---|
| `<LAN_HOST_IP>:80` | TCP | `<REVERSE_PROXY_IP>:80` | Nginx HTTP |
| `<LAN_HOST_IP>:81` | TCP | `<REVERSE_PROXY_IP>:81` | Nginx Proxy Manager Admin |
| `<LAN_HOST_IP>:9443` | TCP | `<REVERSE_PROXY_IP>:443` | HTTPS / Filebrowser entry |
| `<LAN_HOST_IP>:139` | TCP | `<SAMBA_IP>:139` | Samba |
| `<LAN_HOST_IP>:445` | TCP | `<SAMBA_IP>:445` | Samba |

## Core Routing Model

The Proxmox host is the Layer-3 boundary between the LAN and private service network.

IPv4 forwarding is enabled with:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Private-network egress is masqueraded through `<LAN_INTERFACE>`:

```bash
iptables -t nat -A POSTROUTING -s '<PRIVATE_SERVICE_CIDR>' -o <LAN_INTERFACE> -j MASQUERADE
```

Inbound service traffic is selectively DNATed to internal destinations.

## Reverse Proxy / TLS

The Filebrowser public entry point is:

```
https://<LAN_HOST_IP>:9443
```

Traffic follows:

```
Client
  │ HTTPS :9443
  ▼
<LAN_HOST_IP>
  │ DNAT
  ▼
<REVERSE_PROXY_IP>:443
  │ TLS termination
  ▼
Nginx Proxy Manager
  │ HTTP reverse proxy
  ▼
<FILEBROWSER_IP>:8080
  │
  ▼
Filebrowser
```

Filebrowser remains HTTP-only internally while TLS is terminated at the reverse-proxy boundary.

The SSL certificate uses an IP Subject Alternative Name (IP-SAN) for the lab's LAN-facing endpoint.

> **Warning:** The certificate is self-signed and is appropriate for controlled lab infrastructure, not a publicly trusted production service.

## Host Routing Configuration

The relevant `/etc/network/interfaces` rules are:

```
Post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
Post-up   iptables -t nat -A POSTROUTING -s '<PRIVATE_SERVICE_CIDR>' -o <LAN_INTERFACE> -j MASQUERADE
Post-up   iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 139 -j DNAT --to-destination <SAMBA_IP>:139
Post-up   iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 445 -j DNAT --to-destination <SAMBA_IP>:445
Post-up   iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 80 -j DNAT --to-destination <REVERSE_PROXY_IP>:80
Post-up   iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 81 -j DNAT --to-destination <REVERSE_PROXY_IP>:81
Post-up   iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 9443 -j DNAT --to-destination <REVERSE_PROXY_IP>:443
Post-up   iptables -A FORWARD -p tcp --dport 80 -j ACCEPT
Post-up   iptables -A FORWARD -p tcp --dport 81 -j ACCEPT
Post-up   iptables -A FORWARD -p tcp --dport 443 -j ACCEPT
```

## Engineering Highlights

- **Proxmox VE** virtualisation
- Linux Layer-3 routing and bridge networking
- Dual-subnet segmentation
- `iptables` NAT and DNAT
- Stateful traffic-forwarding concepts
- Reverse proxy architecture
- TLS termination and IP-SAN certificates
- Nginx / Nginx Proxy Manager
- Samba network storage
- HTTP payload-size troubleshooting
- Browser-specific TLS troubleshooting
- Infrastructure troubleshooting and root-cause analysis
- Configuration-driven infrastructure

## Design Principles

### Explicit Service Exposure

Each externally accessible service receives an explicit port mapping rather than exposing the entire private network.

### Network Separation

```
<LAN_CIDR>
        │
        │ Proxmox routing boundary
        ▼
<PRIVATE_SERVICE_CIDR>
```

### Centralised TLS Termination

```
Client ── HTTPS ──► Nginx ── HTTP ──► Application
```

### Reproducibility

Network behaviour is declared through host networking configuration rather than relying exclusively on manually configured runtime state.

## Documentation

| Document | Purpose |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Detailed network and traffic-flow architecture |
| [RUNBOOKS/TROUBLESHOOTING.md](RUNBOOKS/TROUBLESHOOTING.md) | Troubleshooting cases and RCA records |

## Security / Production Caveats

> **Important:** This is a home-lab architecture, not a claim of enterprise security compliance.

A hardened production deployment should additionally consider:

- default-deny `FORWARD` policy
- stateful `ESTABLISHED,RELATED` rules
- explicit source-address allowlists
- management-plane isolation
- firewall logging
- certificate lifecycle management
- secrets management
- monitoring and alerting
- configuration versioning
- backup and disaster recovery
- SSH hardening
- VLAN segmentation

The value of this lab is demonstrating practical networking, infrastructure troubleshooting, and the ability to evolve an architecture systematically.

## Public-Portfolio Design

The repository intentionally documents the architecture without publishing live home-network identifiers. The configuration examples use placeholders so the project can be shared safely while remaining technically useful to reviewers.
