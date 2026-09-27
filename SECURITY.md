# Security & Privacy

## Public Repository Policy

This repository is designed to be safe to publish as a technical portfolio.

It intentionally contains **sanitised examples** rather than live deployment state.

### Never commit

- passwords
- API tokens
- SSH private keys
- TLS private keys
- real certificates
- `.env` files containing secrets
- application databases
- authentication/session state
- real usernames where they provide unnecessary personal context
- exact home-network details when they are not required to explain the architecture
- personal filesystem paths
- router credentials or management URLs

### Sanitisation standard

The repository uses placeholders such as:

```text
<LAN_HOST_IP>
<LAN_GATEWAY>
<PRIVATE_SERVICE_IP>
<STORAGE_PATH>
<TIMEZONE>
```

These values are intentionally not the live values used by the lab.

## Current Audit

The repository has been reviewed for common accidental-secret patterns including:

- credentials
- passwords
- tokens
- private keys
- certificate material
- runtime databases

No credentials or private cryptographic material are intentionally included.

## Important distinction

RFC1918 addresses such as `192.168.0.0/16` and `10.0.0.0/8` are not Internet-routable public addresses, but exact home-network details can still reveal information about a private environment.

For that reason, portfolio documentation should prefer **documentation-only addresses and placeholders**.

## Reporting

If a sensitive value is ever committed:

1. Remove it from the working tree.
2. Rotate/revoke the affected credential immediately.
3. Remove it from Git history if it was committed.
4. Verify repository history and forks/caches.
5. Replace it with a secret-management mechanism.

> Removing a secret from the latest commit does not invalidate a secret that was already exposed. Rotation is the important control.
