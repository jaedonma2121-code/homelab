# Troubleshooting Runbooks

This document records significant architectural problems encountered during development of the home lab.

Each incident follows:

1. **The Problem**
2. **Root Cause Analysis (RCA)**
3. **The Engineering Solution**

The objective is to document the reasoning used to identify and resolve infrastructure failures.

---

# Case 1 — Router LAN DNS Constraints

## The Problem

The initial service-exposure model relied too heavily on the router's native LAN/DNS capabilities.

The router did not provide sufficient flexibility to cleanly expose the required services using the desired combination of:

- IP-based access
- non-standard ports
- multiple internal services
- reverse-proxy routing
- direct access to the Proxmox host address

This created a mismatch between the desired service architecture and the capabilities of the consumer-grade router.

The desired model was effectively:

```
LAN Client
    │
    │ service request
    ▼
Router DNS / Port Mapping
    │
    ▼
Internal service
```

## Root Cause Analysis (RCA)

The fundamental problem was architectural coupling between application exposure and the router's DNS/port-forwarding implementation.

The router was being treated as both:

```
DNS / Service Discovery
```

and:

```
Network Traffic Gateway
```

This created limitations when services required different ports and reverse-proxy behaviour.

The lab already had a capable Linux host that could perform:

- routing
- NAT
- DNAT
- port-based service mapping
- reverse proxying

Therefore, continuing to depend on router-specific application behaviour would have increased complexity without providing additional architectural value.

## The Engineering Solution

The architecture was redesigned around an explicit **IP/Port reverse-proxy mapping model**.

Instead of relying on the router to understand each application, the Proxmox host became the deterministic network boundary.

```
LAN
 │
 │ <LAN_HOST_IP>:<PORT>
 ▼
Proxmox
 │
 │ iptables DNAT
 ▼
Private Service
```

| External Endpoint | Internal Destination |
|---|---|
| `<LAN_HOST_IP>:80` | `<REVERSE_PROXY_IP>:80` |
| `<LAN_HOST_IP>:81` | `<REVERSE_PROXY_IP>:81` |
| `<LAN_HOST_IP>:<TLS_PORT>` | `<REVERSE_PROXY_IP>:443` |
| `<LAN_HOST_IP>:139` | `<SAMBA_IP>:139` |
| `<LAN_HOST_IP>:445` | `<SAMBA_IP>:445` |

The responsibility model became:

| Component | Responsibility |
|---|---|
| Router | Upstream LAN connectivity |
| Proxmox | Routing / NAT / DNAT |
| Nginx | HTTP / TLS / reverse proxy |
| Applications | Application functionality |

### Result

Application exposure no longer depended on router-specific DNS or service-routing features.

---

# Case 2 — Safari Strict SSL Handshake Failure

## The Problem

The Filebrowser endpoint was intended to be accessible using:

```
https://<LAN_HOST_IP>:<TLS_PORT>
```

The service worked with some clients, but Safari encountered an SSL/TLS handshake failure.

The architecture was:

```
Safari
   │
   │ TLS
   ▼
<LAN_HOST_IP>:<TLS_PORT>
   │
   │ DNAT
   ▼
Nginx :443
   │
   ▼
Filebrowser :8080
```

Network connectivity was functioning and the internal Filebrowser service was healthy.

## Root Cause Analysis (RCA)

The investigation moved down the connection stack:

```
Application
    ↓
HTTP
    ↓
TLS
    ↓
TCP
    ↓
NAT
    ↓
Network
```

The endpoint was unusual because it used:

```
IP address + non-standard external port + TLS
```

rather than:

```
hostname + TCP/443 + TLS
```

With multiple SSL listeners/configurations, relying on implicit/default server selection introduced ambiguity.

Safari's stricter TLS handling exposed the configuration mismatch.

## The Engineering Solution

An explicit SSL default listener was added to the advanced Nginx configuration:

```nginx
listen <TLS_LISTENER_PORT> ssl default_server;
```

This made the intended listener explicit rather than relying on implicit virtual-server selection.

The certificate was configured with an IP SAN for:

```
<LAN_HOST_IP>
```

The resulting flow became:

```
Safari
   │
   │ HTTPS
   ▼
<LAN_HOST_IP>:<TLS_PORT>
   │
   │ DNAT
   ▼
Nginx
   │
   │ explicit SSL default listener
   │ certificate for <LAN_HOST_IP>
   ▼
TLS session established
   │
   ▼
Reverse proxy
   │
   ▼
Filebrowser
```

## Engineering Principle

A successful TCP connection does not guarantee a successful TLS session.

The troubleshooting sequence was:

1. Verify TCP reachability.
2. Verify NAT/DNAT.
3. Verify Nginx listener.
4. Verify TLS certificate.
5. Verify SAN.
6. Verify server selection.
7. Verify upstream proxying.
8. Verify application response.

### Result

The advanced Nginx configuration explicitly controlled the SSL listener:

```nginx
listen 9443 ssl default_server;
```

This removed ambiguity from the direct-IP TLS endpoint and resolved the Safari handshake mismatch.

> **Operational lesson:** Treat listener selection, SNI, certificate SANs, and reverse-proxy configuration as separate TLS failure domains.

---

# Case 3 — Filebrowser Payload Upload Crash

## The Problem

Filebrowser experienced crashes/failures when uploading larger payloads through the Nginx reverse proxy.

The application worked when accessed internally, but uploads through the externally exposed endpoint failed.

The traffic path was:

```
Client
  │
  │ HTTPS
  ▼
Nginx
  │
  │ HTTP proxy
  ▼
Filebrowser
```

This suggested that the reverse proxy could be imposing a request-body limitation before the payload reached the application.

## Root Cause Analysis (RCA)

HTTP reverse proxies commonly enforce request-body size limits.

The effective request path was:

```
Client
   │
   │ POST / upload
   ▼
Nginx
   │
   │ request-body processing
   ▼
Filebrowser
```

A request could therefore fail before Filebrowser had an opportunity to process the complete upload.

The failure domain was the reverse-proxy request policy rather than necessarily Filebrowser storage or application logic.

## The Engineering Solution

The Nginx request-body limit was removed for the Filebrowser proxy configuration:

```nginx
client_max_body_size 0;
```

The resulting request path became:

```
Client
   │
   │ Large HTTP payload
   ▼
Nginx
   │
   │ client_max_body_size 0
   ▼
Filebrowser
   │
   ▼
Storage
```

### Result

Filebrowser could receive large upload payloads through the reverse proxy without Nginx rejecting them due to its request-body limit.

> **Operational warning:** `client_max_body_size 0` disables Nginx's configured request-body size limit. In an enterprise deployment, an explicit maximum should generally be selected based on storage, bandwidth, workload, timeout, and abuse-protection requirements.

---

# Cross-Incident Engineering Lessons

These incidents demonstrate that application availability depends on several infrastructure layers:

```
┌───────────────────────────────┐
│ Application                   │
├───────────────────────────────┤
│ Reverse Proxy / HTTP          │
├───────────────────────────────┤
│ TLS / Certificate / SNI       │
├───────────────────────────────┤
│ NAT / Firewall / Forwarding   │
├───────────────────────────────┤
│ Linux Networking              │
├───────────────────────────────┤
│ Virtualisation / Bridge       │
├───────────────────────────────┤
│ Physical Network              │
└───────────────────────────────┘
```

A failure at any layer can manifest as a seemingly unrelated application problem.

---

# Troubleshooting Methodology

When diagnosing a service failure, work from the bottom of the stack upward.

## 1. Physical / Interface

```bash
ip addr
ip link
```

Verify:

- `<LAN_INTERFACE>`
- `vmbr0`
- expected IP addresses
- interface state

## 2. Routing

```bash
ip route
```

Confirm routes between:

```
<LAN_CIDR>
<PRIVATE_SERVICE_CIDR>
```

## 3. IP Forwarding

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Expected:

```
1
```

## 4. NAT Rules

```bash
iptables -t nat -L -n -v
```

Inspect:

- `PREROUTING`
- `POSTROUTING`

Packet counters should increase when traffic is generated.

## 5. Forwarding

```bash
iptables -L FORWARD -n -v
```

Confirm that traffic is being accepted and counters increment.

## 6. Listening Services

```bash
ss -lntp
```

Expected listeners include:

```
<SAMBA_IP>:139
<SAMBA_IP>:445
<FILEBROWSER_IP>:8080
<REVERSE_PROXY_IP>:80
<REVERSE_PROXY_IP>:81
<REVERSE_PROXY_IP>:443
```

## 7. Nginx Configuration

Validate before applying changes:

```bash
nginx -t
```

Inspect:

- `listen`
- `server_name`
- `ssl`
- `proxy_pass`
- `client_max_body_size`

## 8. TLS

For direct-IP HTTPS testing:

```bash
openssl s_client \
  -connect <LAN_HOST_IP>:<TLS_PORT> \
  -showcerts
```

Inspect:

- certificate chain
- SAN
- negotiated protocol
- negotiated cipher
- certificate subject
- certificate validity

## 9. Backend Connectivity

Test Filebrowser directly from the private network:

```bash
curl http://<FILEBROWSER_IP>:8080
```

This separates Filebrowser failure from Nginx/TLS/NAT failure.

---

# Incident Resolution Pattern

The general diagnostic model is:

```
External Failure
       │
       ▼
Can the client reach the host?
       │
       ▼
Is NAT working?
       │
       ▼
Is forwarding working?
       │
       ▼
Is the destination listening?
       │
       ▼
Is TLS negotiating?
       │
       ▼
Is Nginx selecting the correct server?
       │
       ▼
Is the reverse proxy forwarding?
       │
       ▼
Is the application processing the request?
```

This prevents prematurely modifying application configuration when the actual fault exists in the infrastructure layer.

---

# Final Operational Principle

> **Troubleshoot from the network boundary toward the application, and isolate each layer before changing configuration.**

The incidents documented here demonstrate practical infrastructure engineering:

- identify the failing layer
- establish a reproducible failure
- inspect the traffic path
- isolate the root cause
- make the smallest architectural change necessary
- validate the result
- document the resolution
- identify production implications of the workaround
