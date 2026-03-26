# Security Release Process

This document defines how theoremOS handles security-impacting releases.

## Severity Classes

- `critical`
- `high`
- `medium`
- `low`

## Publication Rules

- Security-impacting releases still require signed metadata, release notes, conformance evidence, and release-session validation.
- Critical fixes may bypass feature batching, but they may not bypass release validation or signature requirements.
- Superseded preview lines do not receive backport guarantees.

## Advisory Surface

- Security-impacting releases are announced through the public release notes, manifest metadata, integrity evidence, and revocation ledger when necessary.

## Key Rotation

- Release metadata signing keys rotate by publishing a new `releases/trust-root.json` and signing the next release session with the replacement key.
- Revoked or retired keys must remain visible in the trust root with explicit status.
