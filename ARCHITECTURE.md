# Architecture

## 1. Architectural Intent

This home lab implements a two-network service architecture using a Proxmox VE host as the routing, NAT, and traffic-forwarding boundary.

```
                 Upstream LAN
               <LAN_CIDR>
                      │
               <LAN_INTERFACE> / <LAN_HOST_IP>
                      │
             ┌────────▼────────┐
             │  Proxmox Host   │
             │ Routing / NAT   │
             │ DNAT / Forward │
             └────────┬────────┘
                      │
                 vmbr0
              <PRIVATE_GATEWAY_IP>/24
                      │
              Private Services
              <PRIVATE_SERVICE_CIDR>
                      │
          ┌───────────┼───────────┐
          │           │           │
       Samba     Filebrowser     NPM
     <SAMBA_IP>       <FILEBROWSER_IP>      <REVERSE_PROXY_IP>
```

The topology separates external/LAN connectivity, host-level packet processing, and internal application services.

---

## 2. Dual-Subnet Design

### 2.1 LAN / Wi-Fi Subnet

```
<LAN_CIDR>
```

The Proxmox host connects to the upstream network through:

```
Interface: <LAN_INTERFACE>
Address:   <LAN_HOST_IP>/24
```

This is the LAN-facing interface for services deliberately mapped through the host.

### 2.2 Private Bridge Subnet

Internal services use:

```
<PRIVATE_SERVICE_CIDR>
```

The Linux bridge is:

```
vmbr0
<PRIVATE_GATEWAY_IP>/24
```

| Address | Component |
|---|---|
| `<PRIVATE_GATEWAY_IP>` | Proxmox / vmbr0 |
| `<SAMBA_IP>` | Samba |
| `<FILEBROWSER_IP>` | Filebrowser |
| `<REVERSE_PROXY_IP>` | Nginx Proxy Manager |

The private network acts as the service plane.

---

## 3. Routing Boundary

The Proxmox host performs Layer-3 forwarding between `<LAN_CIDR>` and `<PRIVATE_SERVICE_CIDR>`.

IPv4 forwarding is explicitly enabled:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

This changes the host from a conventional endpoint into a packet-forwarding node.

---

## 4. Outbound NAT

Traffic originating from the private network is masqueraded when leaving through `<LAN_INTERFACE>`:

```bash
iptables -t nat -A POSTROUTING \
  -s '<PRIVATE_SERVICE_CIDR>' \
  -o <LAN_INTERFACE> \
  -j MASQUERADE
```

Traffic flow:

```
<PRIVATE_SERVICE_IP>
    │
    ▼
vmbr0
    │
    ▼
Proxmox routing stack
    │
    │ POSTROUTING / MASQUERADE
    ▼
<LAN_INTERFACE>
<LAN_HOST_IP>
    │
    ▼
Upstream LAN
```

The source address is translated from the private address to the host's LAN address.

---

## 5. Inbound DNAT

Inbound traffic arriving through `<LAN_INTERFACE>` is selectively translated using the `PREROUTING` chain.

### Samba

```bash
iptables -t nat -A PREROUTING \
  -i <LAN_INTERFACE> \
  -p tcp \
  --dport 139 \
  -j DNAT \
  --to-destination <SAMBA_IP>:139

iptables -t nat -A PREROUTING \
  -i <LAN_INTERFACE> \
  -p tcp \
  --dport 445 \
  -j DNAT \
  --to-destination <SAMBA_IP>:445
```

Traffic therefore follows:

```
<LAN_CLIENT_IP>:445
       │
       ▼
<LAN_HOST_IP>:445
       │
       │ DNAT
       ▼
<SAMBA_IP>:445
```

---

## 6. Nginx Proxy Manager Traffic

HTTP:

```bash
iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 80 \
  -j DNAT --to-destination <REVERSE_PROXY_IP>:80
```

Administrative UI:

```bash
iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 81 \
  -j DNAT --to-destination <REVERSE_PROXY_IP>:81
```

HTTPS/Filebrowser entry:

```bash
iptables -t nat -A PREROUTING -i <LAN_INTERFACE> -p tcp --dport 9443 \
  -j DNAT --to-destination <REVERSE_PROXY_IP>:443
```

Resulting traffic path:

```
Client
  │
  │ HTTPS :9443
  ▼
<LAN_HOST_IP>
  │
  │ DNAT
  ▼
<REVERSE_PROXY_IP>:443
  │
  │ TLS termination
  ▼
Nginx Proxy Manager
  │
  │ HTTP reverse proxy
  ▼
<FILEBROWSER_IP>:8080
  │
  ▼
Filebrowser
```

---

## 7. Forwarding Rules

The supplied host configuration explicitly permits forwarded HTTP/HTTPS traffic:

```bash
iptables -A FORWARD -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -p tcp --dport 81 -j ACCEPT
iptables -A FORWARD -p tcp --dport 443 -j ACCEPT
```

These rules operate in the `FORWARD` chain after routing determines that traffic is traversing the host.

> **Operational note:** These rules do not contain source restrictions or connection-state matching. If the host's `FORWARD` policy is permissive, additional traffic may remain routable. A hardened deployment should use a default-deny forwarding policy with explicit destination/source and `ESTABLISHED,RELATED` rules.

---

## 8. Reverse Proxy Architecture

The reverse proxy prevents every application from needing its own externally exposed TLS configuration.

```
LAN-facing endpoint
       │
       ▼
     Nginx
       │
       ▼
Internal application
```

Nginx centralises:

- TLS termination
- certificate handling
- HTTP routing
- proxy headers
- request-size policy
- connection handling
- application exposure

---

## 9. Direct IP TLS Endpoint

The Filebrowser endpoint uses an external TLS port represented by `<TLS_PORT>`.

The public endpoint uses an IP address rather than a DNS hostname.

The TLS certificate therefore contains:

```
IP SAN:
<LAN_HOST_IP>
```

Connection flow:

```
TCP connection
      │
      ▼
<LAN_HOST_IP>:9443
      │
      │ DNAT
      ▼
<REVERSE_PROXY_IP>:443
      │
      ▼
Nginx TLS listener
```

The certificate presented by Nginx must identify the IP address used by the client.

---

## 10. Nginx `default_server` Behaviour

For the direct-IP endpoint, an explicit SSL default listener is used:

```nginx
listen <TLS_LISTENER_PORT> ssl default_server;
```

This gives the dedicated endpoint a deterministic SSL server configuration.

```
Client
   │
   │ TLS handshake
   ▼
<LAN_HOST_IP>:9443
   │
   ▼
Nginx default SSL server
   │
   │ Certificate / TLS negotiation
   ▼
Client
   │
   │ HTTP over established TLS
   ▼
Reverse proxy
   │
   ▼
<FILEBROWSER_IP>:8080
```

This became particularly important when Safari testing exposed a TLS listener/handshake mismatch.

---

## 11. TLS Handshake Architecture

The connection can be viewed as three layers.

### Layer 1 — TCP

```
Client → <LAN_HOST_IP>:9443
```

### Layer 2 — TLS

```
Client
  │
  │ ClientHello
  ▼
Nginx
  │
  │ Certificate / ServerHello
  ▼
Client
```

### Layer 3 — HTTP Proxy

After successful TLS negotiation:

```
Client
  │ HTTPS
  ▼
Nginx
  │ HTTP
  ▼
Filebrowser :8080
```

Filebrowser does not need to understand the external TLS session.

---

## 12. Why Filebrowser Remains HTTP Internally

Filebrowser listens on:

```
<FILEBROWSER_IP>:8080
```

and is HTTP-only.

The encrypted trust boundary is placed at the reverse proxy:

```
                    TLS boundary
                         │
Client ── HTTPS ──► Nginx ── HTTP ──► Filebrowser
```

For a higher-security deployment, TLS could also be introduced between the reverse proxy and backend application.

---

## 13. End-to-End Traffic Maps

### Filebrowser

```
Client
  │
  │ HTTPS :9443
  ▼
<LAN_HOST_IP>
  │
  │ DNAT
  ▼
<REVERSE_PROXY_IP>:443
  │
  │ TLS termination
  ▼
Nginx Proxy Manager
  │
  │ HTTP
  ▼
<FILEBROWSER_IP>:8080
  │
  ▼
Filebrowser
```

### Samba

```
Client
  │
  │ TCP :445
  ▼
<LAN_HOST_IP>
  │
  │ DNAT
  ▼
<SAMBA_IP>:445
  │
  ▼
Samba
```

---

## 14. Failure Domains

| Failure Domain | Example Failure |
|---|---|
| LAN | Client cannot reach `<LAN_HOST_IP>` |
| Host interface | `<LAN_INTERFACE>` unavailable |
| Routing | IPv4 forwarding disabled |
| NAT | Incorrect `PREROUTING` / `POSTROUTING` rule |
| Bridge | `vmbr0` unavailable |
| Reverse proxy | Nginx listener/configuration failure |
| TLS | Certificate/SNI/handshake mismatch |
| Application | Filebrowser/Samba unavailable |
| HTTP proxy | Request body / timeout / header restrictions |

This separation allows troubleshooting to progress from lower-level connectivity toward application-level behaviour.

---

## 15. Architectural Trade-offs

### Benefits

- Clear network segmentation
- Centralised traffic management
- Explicit port exposure
- Centralised TLS termination
- Application isolation
- Reproducible host networking configuration
- Practical Linux networking experience

### Trade-offs

- The Proxmox host becomes a critical routing dependency.
- Wi-Fi is being used as the physical uplink.
- Host-level `iptables` rules increase operational complexity.
- Self-signed certificates require client-side trust handling.
- Direct IP HTTPS endpoints are less flexible than DNS-based hostnames.
- Nginx Proxy Manager becomes part of the application access path.

---

## 16. Security Considerations

Recommended future improvements include:

```
Default-deny FORWARD policy
        │
        ├── Explicit allow rules
        ├── Stateful connection tracking
        ├── Source subnet restrictions
        ├── Management-plane isolation
        ├── Firewall logging
        └── Monitoring / alerting
```

Additional improvements could include:

- VLAN segmentation
- dedicated physical Ethernet uplink
- dedicated management VLAN
- DNS-based service discovery
- trusted internal CA
- automated certificate renewal
- secrets management
- configuration automation
- GitOps-style deployment
- centralised logging
- network monitoring

## 17. Architecture Summary

The key principle is:

> **Expose services deliberately at the network boundary while keeping application workloads on an isolated private service network.**

The Proxmox host acts simultaneously as:

```
Virtualisation Platform
        +
Linux Network Boundary
        +
NAT Gateway
        +
DNAT / Port Forwarder
```

while Nginx provides:

```
TLS Termination
        +
HTTP Reverse Proxy
        +
Application Exposure Layer
```


## 15. Port Translation Note

The host-level NAT example may translate an external TLS port to the reverse proxy's native HTTPS port. In that model, the Nginx listener inside the private network is the destination port (typically 443), not the external port. If Nginx itself listens on the external port, the DNAT rule must preserve that port.

This distinction is documented deliberately because port translation and Nginx listener selection are separate layers.


## 15. Port Translation Note

Port translation and Nginx listener selection are separate layers. If the host translates an external TLS port to the reverse proxy's native HTTPS port, Nginx receives the destination port after DNAT. Conversely, if Nginx is configured to listen on the external port itself, the DNAT rule must preserve that port.

This distinction is important when documenting or troubleshooting direct-IP TLS endpoints.
