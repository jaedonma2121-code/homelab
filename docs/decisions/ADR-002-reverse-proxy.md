# ADR-002: Centralise HTTP and TLS at the Reverse Proxy

- **Status:** Accepted
- **Decision:** Terminate external TLS at Nginx and proxy HTTP traffic to internal applications.

## Context

Individual applications should not each need separate externally exposed TLS configuration.

## Decision

Use:

```text
Client
  |
  | HTTPS
  v
Nginx
  |
  | HTTP
  v
Internal Application
```

The reverse proxy owns:

- TLS termination
- certificate selection
- HTTP routing
- request-size policy
- proxy headers
- application exposure

## Consequences

### Positive

- Centralised certificate management
- Consistent ingress architecture
- Reduced application-facing exposure
- Easier troubleshooting

### Negative

- Nginx becomes part of the service dependency chain.
- Internal traffic remains HTTP unless backend TLS is separately enabled.
- Reverse-proxy limits can affect application behaviour.

## Future Evolution

Introduce backend TLS or mTLS where workload sensitivity justifies the added complexity.
