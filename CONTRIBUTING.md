# Contributing

This repository is primarily a personal infrastructure portfolio, but the engineering practices are intentionally collaborative.

## Changes

Prefer small, focused commits using conventional prefixes:

- `docs:` documentation
- `feat:` new infrastructure capability
- `fix:` correction
- `security:` security hardening
- `refactor:` structural improvement
- `chore:` maintenance

## Validation

Before committing infrastructure changes:

1. Check configuration syntax.
2. Review network exposure.
3. Confirm no credentials or local-only data were introduced.
4. Update the relevant architecture/runbook documentation.
5. Test the change in the lab before describing it as validated.

## Privacy

Do not commit live credentials, private keys, application databases, personal filesystem paths, or unnecessary details about the physical home network.
