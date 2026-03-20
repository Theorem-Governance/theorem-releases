# Release Contract

This document defines the supported release contract for `theorem-node`
artifacts published from the Ring 1 release host.

## Scope

This contract exists so deployment automation can rely on stable release output
without reverse-engineering the release tooling.

## Asset Naming

Binary release assets use this naming pattern:

- `theorem-node-<tag>-<target>.tar.gz`
- `theorem-node-<tag>-<target>.tar.gz.sha256`

Examples:

- `theorem-node-v0.3.0-x86_64-unknown-linux-gnu.tar.gz`
- `theorem-node-v0.3.0-x86_64-unknown-linux-gnu.tar.gz.sha256`

The tarball expands into a directory with the same base name:

- `theorem-node-<tag>-<target>/`

## Checksum Contract

Each release tarball has a sibling SHA-256 checksum file with the exact same
base name plus `.sha256`.

Verification example:

```bash
shasum -a 256 -c theorem-node-v0.3.0-x86_64-unknown-linux-gnu.tar.gz.sha256
```

## SBOM Contract

Each tagged release includes a CycloneDX SBOM generated from the Rust workspace.
The SBOM is published as a release asset alongside the binaries and checksums.

## Supported Targets

The intended target set is:

- `x86_64-unknown-linux-gnu`
- `aarch64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

Linux release publication is authoritative for production deployment and is
allowed to proceed independently of Darwin artifact completion.

## Publish Semantics

- Linux artifacts and the SBOM are the minimum release set required for a tag
  to publish.
- Darwin artifacts may be attached after the Linux release is already live.
- A Darwin build failure must not block publication of Linux artifacts.

## Automation Guidance

- Automate against explicit tags such as `v0.3.0`.
- Do not treat the GitHub "latest" pointer as a stable deployment contract.
- Before promoting a release, verify the checksum for the exact asset you
  downloaded.

## Canonical Release Authority Path

Release publication and release authority are separate concerns.

The canonical theorem-native authority path for a runtime artifact is:

1. Admit the artifact hash with `POST /admit-artifact` or `theorem-node admit-artifact`.
2. Verify the candidate hash with `POST /verify-release` or `theorem-node verify-release`.
3. Use the verification result or a theorem-issued execution capability as the
   gate for Ring 3 runtime admission.

`POST /verify-release` is part of the stable public authority contract.
`GET /export-state`, `GET /rebuild-authority-state`, and
`POST /restore-authority-state` are operator/admin recovery routes, not release
integration APIs.

## Production Install Guidance

For production hosts:

1. Download a tagged release.
2. Verify the `.sha256` file before unpacking.
3. Install into a versioned directory.
4. Point a stable symlink such as `/var/lib/theorem/current` at that release.
5. Use `GET /health` for liveness and `GET /status` for readiness and state
   inspection after restart.
