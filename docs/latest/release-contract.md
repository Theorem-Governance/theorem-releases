# Release Contract

This document defines the supported release contract for `theorem-node`
artifacts published from the Ring 1 release host and the public machine
contract line expected to remain stable through `3.x`.

## Scope

This contract exists so deployment automation and downstream theorem-native
systems can rely on stable release output without reverse-engineering tooling,
reading prose diffs, or compensating with shell logic outside Theorem.

## Versioned Upgrade Contract

Public `3.x` releases must classify compatibility explicitly.

Compatibility classes:

- `compatible-additive`: adds new machine capabilities without removing or
  silently changing existing stable behavior
- `compatible-behavior-tightening`: preserves the same stable surface but
  changes validation, enforcement, or default behavior in a way downstream
  consumers must understand
- `breaking-major`: removes or incompatibly changes stable machine behavior and
  requires a new major version line

Rules:

- breaking removals and incompatible semantic changes belong on a major version
  boundary, not a minor release
- deprecated machine-consumable surfaces remain available for at least one full
  minor release after deprecation is published in a release manifest
- every public release must declare its compatibility classification in a
  machine-readable manifest

## Asset Naming

Binary release assets use this naming pattern:

- `theorem-node-<tag>-<target>.tar.gz`
- `theorem-node-<tag>-<target>.tar.gz.sha256`

Examples:

- `theorem-node-v3.0.0-x86_64-unknown-linux-gnu.tar.gz`
- `theorem-node-v3.0.0-x86_64-unknown-linux-gnu.tar.gz.sha256`

The tarball expands into a directory with the same base name:

- `theorem-node-<tag>-<target>/`

## Checksum Contract

Each release tarball has a sibling SHA-256 checksum file with the exact same
base name plus `.sha256`.

Verification example:

```bash
shasum -a 256 -c theorem-node-v3.0.0-x86_64-unknown-linux-gnu.tar.gz.sha256
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

## Machine-Readable Release Manifest

Every public release must ship a structured manifest with:

- version
- release date
- compatibility classification
- new, changed, deprecated, and removed capabilities
- schema changes
- API additions and removals
- required operator actions
- release artifact metadata

Contract files:

- schema: `docs/release-manifest.schema.json`
- input template: `docs/release-manifest-input.template.json`
- release asset name: `theorem-release-manifest-<tag>.json`
- public manifest copy: `releases/<tag>/manifest.json`

Release publication must fail closed if manifest metadata is missing. The
manifest is not inferred from git history.

## Public Release Feed

Every public release must also update the machine-readable release feed:

- feed path: `releases/feed.json`
- each item includes version, manifest path, human release notes URL, release
  URL, compatibility classification, and conformance report path when present

This feed is the stable discovery surface for public releases. Consumers should
not scrape HTML or infer deltas from prose pages.

## Conformance Evidence

`3.0` publication also carries a machine-readable conformance report when
available.

Contract files:

- conformance runner: `scripts/run-3.0-conformance.ps1`
- public conformance copy: `releases/<tag>/conformance.json`

The conformance report is the published proof-of-product for the machine
contract. If a required `3.0` user moment is still missing from executable
coverage, the report must say so explicitly.

## Operator Demonstration Path

`3.0` publication also carries a machine-readable demonstration path artifact.

Contract files:

- demo path generator: `scripts/generate-3.0-demo-path.ps1`
- public demo path copy: `releases/<tag>/demo-path.json`

The demonstration path captures the ordered operator proof steps that establish
the `3.0` user moments and the governed release act.

## Governed Release Declaration

Public release publication must also emit a theorem-native governed release act.

Contract surface:

- query route: `GET /releases/declared`
- record route: `POST /releases/declared`

Minimum declaration fields:

- version
- scope
- declaring entity
- manifest path
- conformance report path when present
- release URL
- evidence summary

The publish flow must fail closed if it cannot record the declaration. Release
metadata and artifacts are not considered complete `3.x` publication until the
queryable release declaration exists.

## Automation Guidance

- Automate against explicit tags such as `v3.0.0`.
- Do not treat the GitHub "latest" pointer as a stable deployment contract.
- Before promoting a release, verify the checksum for the exact asset you
  downloaded.
- Consume `releases/feed.json` and release manifests instead of scraping release
  notes.

## Canonical Release Authority Path

Release publication and release authority are separate concerns.

The canonical theorem-native authority path for a runtime artifact is:

1. Admit the artifact hash with `POST /admit-artifact` or `theorem-node admit-artifact`.
2. Verify the candidate hash with `POST /verify-release` or `theorem-node verify-release`.
3. Query `GET /runtime-eligibility/artifact` with the candidate hash and
   runtime entity/scope to obtain the trust-aware eligibility answer used by
   theorem-supervisor and other runtime consumers.
4. Use theorem-issued execution capabilities only after the trust-aware runtime
   eligibility path is satisfied.

`POST /verify-release` is part of the stable public authority contract.
`GET /export-state`, `GET /rebuild-authority-state`, and
`POST /restore-authority-state` are operator/admin recovery routes, not release
integration APIs. `GET /runtime-eligibility/artifact` is an operator/admin
runtime-control surface rather than a public integration endpoint.

## Production Install Guidance

For production hosts:

1. Download a tagged release.
2. Verify the `.sha256` file before unpacking.
3. Read the release manifest for compatibility classification and required
   operator actions.
4. Install into a versioned directory.
5. Point a stable symlink such as `/var/lib/theorem/current` at that release.
6. Use `GET /health` for liveness and `GET /status` for readiness and state
   inspection after restart.
