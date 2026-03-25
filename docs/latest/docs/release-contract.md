# Release Contract

This document defines the supported release contract for `theorem-node`
artifacts published from the Ring 1 release host and the public machine
contract line expected to remain stable through `0.5.x`.

## Scope

This contract exists so deployment automation and downstream theorem-native
systems can rely on stable release output without reverse-engineering tooling,
reading prose diffs, or compensating with shell logic outside Theorem.

## Definitions

- **staged candidate**: A release that has passed conformance validation but
  has not yet been published to the public release feed. Staged candidates
  exist only in the build pipeline and are not visible to downstream consumers
  until publication completes.

## Versioned Upgrade Contract

Public `0.5.x` releases must classify compatibility explicitly.

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

### Single-Binary Tarball (backward compat)

Binary release assets use this naming pattern:

- `theorem-node-<tag>-<target>.tar.gz`
- `theorem-node-<tag>-<target>.tar.gz.sha256`

Examples:

- `theorem-node-v0.5.0-x86_64-unknown-linux-gnu.tar.gz`
- `theorem-node-v0.5.0-x86_64-unknown-linux-gnu.tar.gz.sha256`

The tarball expands into a directory with the same base name:

- `theorem-node-<tag>-<target>/`

### Rootfs Tarball (autonomous deployment)

FreeBSD rootfs tarballs contain all theorem binaries, configuration files, the
installer script, and the first-boot provisioning example. This is the
recommended artifact for autonomous downstream deployment.

Naming pattern:

- `theoremos-<tag>-<target>.tar.gz`
- `theoremos-<tag>-<target>.tar.gz.sha256`

Examples:

- `theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz`
- `theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256`

The tarball expands into `theoremos-<tag>-<target>/` with this layout:

```
sbin/theorem-init                          PID 1 supervisor
usr/local/bin/theorem-node                 Governance kernel
usr/local/bin/theorem-bus                  Message bus
usr/local/bin/theorem-gateway              API gateway
usr/local/bin/theorem-supervisor           Process supervisor
usr/local/bin/theorem-govagent             Governance agent
usr/local/bin/theorem-witness              Witness node
usr/local/etc/rc.d/theorem_init            FreeBSD rc.d service script
etc/rc.conf                                FreeBSD system configuration
etc/theorem/theorem.conf                   Default governance config
etc/veriexec.d/theorem.manifest            Binary integrity manifest
etc/theorem/first-boot.conf.example        First-boot provisioning template
boot/kernel/THEOREMOS                      Custom kernel config
install.sh                                 Non-interactive installer
```

### Install Path

For autonomous deployment from the rootfs tarball:

```bash
tar xzf theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz
cd theoremos-v0.5.0-x86_64-unknown-freebsd
# Optional: cp etc/theorem/first-boot.conf.example /tmp/first-boot.conf && edit
FIRST_BOOT_CONF=/tmp/first-boot.conf sh install.sh
```

The installer verifies FreeBSD >= 14, installs binaries, applies first-boot
configuration, sets immutable flags, and enables the theorem-init service.

Machine consumers must distinguish the FreeBSD artifact classes:

- `theoremos-<tag>-x86_64-unknown-freebsd.tar.gz` is the installer-capable
  rootfs artifact. Its manifest entry carries `artifact_kind=installer-rootfs`,
  `automation_strategy=extract-and-run-installer`, `archive_root`, an
  `installer_entrypoint` relative to that archive root, and
  `direct_disk_write=forbidden`.
- `theoremos-<tag>-amd64.iso` and `theoremos-<tag>-amd64-memstick.img` are
  install media only. Their manifest entries carry
  `artifact_kind=install-media` and `direct_disk_write=forbidden`.
- Automation must fail closed if no `installer-rootfs` or future
  `direct-write-disk-image` artifact is published for the requested release.

See `docs/theorem-init-contract.md` for the service lifecycle, and
`docs/operator-guide/08-migration-4x-to-5x.md` for the migration path.

## Checksum Contract

Each release tarball has a sibling SHA-256 checksum file with the exact same
base name plus `.sha256`.

Verification example:

```bash
shasum -a 256 -c theorem-node-v0.5.0-x86_64-unknown-linux-gnu.tar.gz.sha256
```

## SBOM Contract

Each tagged release includes a CycloneDX SBOM generated from the Rust workspace.
The SBOM is published as a release asset alongside the binaries and checksums.

NOTE: SBOM and provenance attestation assets are produced by the build
pipeline on the FreeBSD build host. They do not appear in the source repository.
The release script (`generate-release-manifest.ps1`) validates their presence
at publication time and fails closed if they are absent.

## Provenance Contract

Each `0.5.x` tagged release also includes at least one machine-readable
provenance attestation asset. Accepted provenance asset forms are:

- `*.intoto.jsonl`
- `*.provenance.json`
- `*.attestation.json`
- `*.sigstore.json`

Release publication fails closed if a `0.5.x` tag is missing provenance evidence.

## Release Integrity Evidence

NOTE: The `releases/` directory tree referenced throughout this contract is a
publication-time artifact generated by the release pipeline. It does not exist
in the staged candidate or pre-release repository. See
`scripts/publish-release.ps1` for the publication process.

Each `0.5.x` manifest must publish a `release_integrity` object and each release
must attach a separate integrity evidence asset:

- release asset name: `theorem-release-integrity-<tag>.json`
- public copy: `releases/<tag>/integrity.json`
- schema: `docs/release-integrity.schema.json`

The integrity evidence is part of the shipped product contract. It must carry:

- SBOM asset references
- provenance asset references
- release ring
- minimum-version downgrade-prevention rule
- rollback authorization rule
- revocation ledger and per-release record paths
- mirror expiry with fail-closed stale behavior

## Recovery Contract

Each `0.5.x` release line must also make recovery answerable from shipped product
artifacts rather than maintainer memory. Recovery is lawful only when a machine
returns from an approved recovery image plus theorem-native replay to the same
authority state it held before loss.

Minimum recovery contract fields:

- approved image digest and recovery image digest
- measured boot evidence that binds the recovery image and theorem runtime
- recovery image rights scoped to machine and approved channel
- replay contract with explicit authority-state digest and event count
- trust-root custody record for the active recovery root set
- anti-replay recovery binding over authority-state digest and chronology
- disconnected verification evidence proving offline validation is possible

The contract standard is authority-state equivalence, not machine liveness
alone.

## Supported Targets

The intended target set is:

- `x86_64-unknown-freebsd` (primary — production target)
- `aarch64-unknown-freebsd` (planned)
- `x86_64-unknown-linux-gnu` (legacy — may be reintroduced)

FreeBSD release publication is authoritative for production deployment.

## Publish Semantics

- FreeBSD artifacts and the SBOMs are the minimum release set required for a
  tag to publish.
- FreeBSD bare-metal automation must target a manifest artifact marked
  `artifact_kind=installer-rootfs` or `artifact_kind=direct-write-disk-image`.
  Install media artifacts marked `artifact_kind=install-media` must not be used
  for unattended disk writes.
- `scripts/validate-release.sh` must pass before any publication. This is a
  machine-enforced gate, not a human-remembered checklist.

## Machine-Readable Release Manifest

Every public release must ship a structured manifest with:

- version
- release date
- compatibility classification
- new, changed, deprecated, and removed capabilities
- explicit capability-negotiation contract
- explicit principal semantics contract
- explicit stable error taxonomy for published `0.5.0` routes
- explicit act timing and resource-consumption surfaces
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
  URL, compatibility classification, release ring, integrity path,
  minimum-version floor, and revocation paths
- the feed itself carries `mirror_generated_at`, `mirror_expires_at`, and
  `stale_behavior`

This feed is the stable discovery surface for public releases. Consumers should
not scrape HTML or infer deltas from prose pages.

## Capability Negotiation

Public `0.5.x` capability negotiation is release-time negotiation, not request
header negotiation.

Downstream consumers negotiate against:

- `releases/feed.json`
- `releases/<tag>/manifest.json`
- the manifest's `compatibility_classification`
- the manifest's `capability_negotiation` section
- the live runtime surfaces `GET /capabilities` and
  `GET /capability-negotiation`

Rules:

- capability ids published in a manifest are stable within the `0.5.x` line once
  released
- consumers must gate on explicit capability ids and compatibility class rather
  than inferring behavior from prose notes or semver alone
- unknown capability ids are additive publication evidence, not permission to
  guess unpublished semantics

The running `0.5.x` server also publishes discovery surfaces:

- `GET /`
- `GET /openapi.json`
- `GET /openapi.yaml`
- `GET /capabilities`
- `GET /capability-negotiation`
- `GET /auth/discovery`

## Principal Semantics

The public contract distinguishes acting/query entities, principals, and
grantors explicitly.

Stable semantics:

- `entity_id` names the acting or queried entity on public request surfaces
- `principal_id` names the entity holding authority on authorization surfaces
- `granted_by_id` names the grantor and must not be interpreted as the
  authority holder
- `GET /authorizations?entity_id=<entity>` returns authorizations held by that
  principal entity, not authorizations granted by it

Every public `0.5.x` manifest must repeat these mappings in
`principal_contract`.

## Stable Error Taxonomy

The stable machine error taxonomy applies to the published `0.5.0` organization,
execution-lifecycle, treaty, intervention, and governed-release routes that use
JSON `api_error(...)` bodies.

Stable body shape for those routes:

- content type: `application/json`
- fields: `error_code`, `message`

Rules:

- published `error_code` values are stable within `0.5.x`
- `409` is the stable idempotency and conflict status for reused keys with a
  different request body
- legacy pre-`0.5.0` routes may still return `text/plain` rejection bodies and
  are outside this stable taxonomy unless and until promoted into it

Every public `0.5.x` manifest must publish the scoped taxonomy in
`stable_error_taxonomy`.

## Runtime Auth Discovery

Protected runtime routes publish their auth contract through
`GET /auth/discovery`.

Current `0.5.x` auth contract:

- scheme: `Bearer`
- header: `Authorization: Bearer <token>`
- environment variable: `THEOREM_WITNESS_TOKEN`
- live schema surface: `GET /openapi.json`

## Act Timing And Resource Surfaces

The public act contract must publish both the timing surfaces that govern act
admission and the resource-consumption surfaces that make cost accounting
rebuildable.

Stable timing surface:

- `POST /declare-intent` returns `declared_at`, `minimum_wait_ms`, and
  `earliest_execution_at`
- `POST /submit-act` returns `initiated_at`, `intent_declared_at`,
  `elapsed_ms_since_declared`, and `resource_consumption_entries_recorded`
- E4 enforcement is defined over the published timing values rather than hidden
  server-local convention

Stable resource-consumption surface:

- `POST /submit-act` accepts `resource_consumption_entries`
- each entry publishes `resource_type`, `quantity`, `unit`,
  `measurement_method_id`, `measured_at`, `within_limits`, and
  `escalation_signal_id`

Every public `0.5.x` manifest must publish these bindings in `act_contract`.

## Conformance Evidence

`0.5.0` publication also carries a machine-readable conformance report when
available.

Contract files:

- conformance runner: `scripts/validate-session-26-public-conformance-and-rehearsal.ps1`
- public conformance copy: `releases/<tag>/conformance.json`

The conformance report is the published proof-of-product for the machine
contract. If a required `0.5.0` user moment is still missing from executable
coverage, the report must say so explicitly.

Contract modules verify internal fixture consistency (cross-field relationships
within the same JSON bundle). They do not verify that fixture field values
correspond to external infrastructure state. This is a deliberate design
boundary: contracts verify structural governance correctness, not runtime truth.

## Operator Demonstration Path

`0.5.0` publication also carries a machine-readable demonstration path artifact.

Contract files:

- public demo path copy: `releases/<tag>/demo-path.json` (generated by the release pipeline)

The demonstration path captures the ordered operator proof steps that establish
the `0.5.0` user moments and the governed release act.

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
metadata and artifacts are not considered complete `0.5.x` publication until the
queryable release declaration exists.

## Release Process Discipline

Building and releasing are structurally separate activities. They must not
happen in the same session, the same script invocation, or the same context.

**Development session:** write code, fix bugs, build, test. End with a
committed, tagged state on `main`.

**Release session:** start fresh. Validate the tagged state against this
contract using `scripts/validate-release.sh`. Assemble artifacts. If
validation fails, STOP. Do not fix and ship in the same session. Return to
a development session, fix, re-tag, and start a new release session.

This separation exists because the same context that fixes a build error
lacks the perspective to catch a missing provenance hash or an empty SBOM
array. The release session's only job is validation and publication.

## Downstream Consumer Contract

See `docs/consumer-contract.md` for the downstream-facing contract: what
consumers can depend on, what is stable, what is provisional, and the
explicit install/upgrade path.

The consumer contract is a separate document because release-contract.md
defines what the *release process* must produce, while consumer-contract.md
defines what a *consumer* can rely on. They are complementary.

## Automation Guidance

- Automate against explicit tags such as `v0.5.0`.
- Do not treat the GitHub "latest" pointer as a stable deployment contract.
- Before promoting a release, verify the checksum for the exact asset you
  downloaded.
- Consume `releases/feed.json` and release manifests instead of scraping release
  notes.
- Fail closed if `releases/feed.json` is past `mirror_expires_at`. Use
  `scripts/assert-release-mirror-freshness.ps1` or equivalent logic.
- For `0.5.x`, verify `releases/<tag>/integrity.json` and consume
  `release_integrity.minimum_version` before admitting or mirroring a release.

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
3. Verify the mirror feed is still fresh and fail closed if it is stale.
4. Read the release manifest for compatibility classification, minimum version,
   rollback authorization, and required operator actions.
5. Review `releases/<tag>/integrity.json` for SBOM, provenance, and revocation
   paths before promotion.
6. Verify the measured-boot and recovery bundle offline before admitting a
   recovery image or replay path.
7. Confirm the recovery bundle proves authority-state equivalence, not only a
   successful reboot.
8. Install into a versioned directory.
9. Point a stable symlink such as `/var/lib/theorem/current` at that release.
10. Use `GET /health` for liveness and `GET /status` for readiness and state
   inspection after restart.
