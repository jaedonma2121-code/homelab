# Security and Public-Repository Policy

This repository is public and therefore treats the live homelab configuration as sensitive infrastructure information.

## Never commit

- Real IP addresses
- Real hostnames or local DNS names
- Public IP addresses
- MAC addresses
- Wi-Fi SSIDs or passwords
- Home-specific usernames
- Passwords, hashes, API keys, tokens, cookies, or session data
- Private Certificate Authority keys
- Server private keys
- Unreviewed live certificate bundles
- Exact DNAT, firewall, or port-forwarding rules
- Physical disk/device identifiers
- Container IDs or VM identifiers
- Application databases
- Backup archives
- Raw logs containing sensitive data
- Unredacted command output from the live infrastructure

## Safe documentation pattern

Use symbolic placeholders such as <LAN_SUBNET>, <SERVICE_SUBNET>, <HOST_ADDRESS>, <SERVICE_ADDRESS>, <REVERSE_PROXY>, <APPLICATION>, and <APP_PORT>.

The purpose is to document architecture and engineering decisions, not publish a live network blueprint.

## Private PKI

A CA certificate and a CA private key are not equivalent. The CA private key is secret and must never be published or installed on client devices.

Server private keys are also secret. Review server certificates before publishing because certificate metadata can reveal hostnames or other identifying information.

Keep private PKI material outside Git.

## Storage

Physical storage should be managed by the hypervisor/host where practical. Containers should receive only the directories they need.

Do not publish physical disk serial numbers, filesystem UUIDs, identifying storage labels, or sensitive host mount information.

## Remote access

Do not make the file-management application publicly reachable merely to enable remote family access.

Prefer:

1. VPN/private overlay network
2. Authenticated access
3. HTTPS
4. Least-privilege application accounts
5. No direct exposure of backend application ports

## Pre-commit review

Search for private-network indicators such as private IPv4 ranges and .local names. These are not automatically secrets, but they should trigger manual review.

Also search for:

~~~text
password
passwd
secret
token
api_key
private_key
BEGIN PRIVATE KEY
BEGIN OPENSSH PRIVATE KEY
~~~

## Incident response

If a secret or sensitive mapping is accidentally committed:

1. Remove it from the working tree.
2. Rotate the affected credential or key.
3. Remove sensitive data from Git history.
4. Check downstream copies where appropriate.
5. Review related infrastructure exposure.

Deleting a secret from the latest commit alone is not sufficient if it remains in Git history.

## Public portfolio objective

The repository should demonstrate architecture, networking, virtualisation, reverse proxying, TLS, storage design, service management, troubleshooting, and security reasoning while withholding information that would help an external party enumerate or target the home environment.
