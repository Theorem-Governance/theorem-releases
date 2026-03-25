# Consumer Contract

This document defines what a downstream consumer of theoremOS can depend on,
what is stable, what is not, and what the install/upgrade/rollback path looks
like. It exists so consumers can adopt theoremOS as a production dependency
without reverse-engineering the release process.

## Current Stage

theoremOS is at the **platform preview** stage as of v0.5.0. This means:

- The substrate technology is real and running on bare metal FreeBSD
- The release evidence chain (manifest, integrity, conformance, provenance,
  SBOMs, revocation) is published and machine-readable
- The FreeBSD rootfs tarball now carries an installer-capable bare-metal
  contract via `install.sh` and `first-boot.conf`
- But: upgrade/rollback paths are not yet machine-executable
- But: not all consumer-facing contracts are explicit

Consumers should add their own fail-closed validation gates until this
contract reaches **production ready** status.

## What You Can Depend On Today

### Stable (will not break within 0.5.x)

- **Release feed**: `releases/feed.json` is the discovery surface. Consume it
  programmatically. It carries freshness and stale_behavior fields — honor
  `fail_closed`.
- **Manifest schema**: `releases/<tag>/manifest.json` declares version,
  compatibility classification, assets, and SBOMs.
- **Integrity schema**: `releases/<tag>/integrity.json` carries SHA-256
  hashes for all release assets.
- **Checksum contract**: Every tarball has a `.sha256` sibling.
- **Revocation ledger**: `releases/revocations.json` tracks active/revoked
  status per version.
- **Asset naming**: `theorem-node-<tag>-<target>.tar.gz`
- **Bare-metal artifact typing**: release manifests distinguish
  `installer-rootfs` from `install-media`; automation must fail closed on
  `install-media`
- **Installer archive contract**: `installer-rootfs` artifacts publish
  `archive_root` plus `installer_entrypoint`; consumers must resolve the
  installer path relative to that archive root

### Provisional (may change shape in 0.6.x)

- **Conformance report**: `releases/<tag>/conformance.json` — test names and
  structure may change as the substrate test suite matures.
- **Provenance format**: Currently `*.provenance.json` — may migrate to
  in-toto or sigstore format.
- **Demo path**: `releases/<tag>/demo-path.json` — operator proof steps,
  schema not yet frozen.
- **HTTP API surface**: The `GET /capabilities`, `GET /health`, `GET /status`
  endpoints exist but response shapes may evolve.

### Not Yet Stable

- **Direct-write disk image**: no machine-published direct-write disk image yet.
  Use the installer-capable rootfs tarball instead of ISO/memstick media.
- **Upgrade path**: No documented or machine-executable upgrade from N-1 to N.
- **Rollback path**: ZFS snapshots exist but no consumer-facing rollback
  contract.
- **Configuration contract**: Config file format and location not yet frozen.

## Supported Targets

| Target | Status |
|--------|--------|
| `x86_64-unknown-freebsd` | Primary. Production builds. |
| `x86_64-unknown-linux-gnu` | Legacy. May be reintroduced. |
| `aarch64-unknown-freebsd` | Planned. Not yet available. |

## How to Consume a Release

### Verification (required)

```bash
# 1. Fetch the feed
curl -sO https://raw.githubusercontent.com/Theorem-Governance/theorem-releases/main/releases/feed.json

# 2. Check freshness — fail if stale
python3 -c "
import json, datetime
feed = json.load(open('feed.json'))
expires = datetime.datetime.fromisoformat(feed['mirror_expires_at'].replace('Z', '+00:00'))
if datetime.datetime.now(datetime.timezone.utc) > expires:
    raise SystemExit('FAIL: feed is stale — expires ' + str(expires))
print('Feed fresh until', expires)
"

# 3. Check revocation status
python3 -c "
import json
revs = json.load(open('releases/revocations.json'))
for r in revs['records']:
    if r['version'] == 'v0.5.0':
        if r['status'] != 'active':
            raise SystemExit('FAIL: version revoked — ' + r['status'])
        print('Version active')
        break
"

# 4. Download and verify binary
curl -LO <release-url>/theorem-node-v0.5.0-x86_64-unknown-freebsd.tar.gz
curl -LO <release-url>/theorem-node-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256
shasum -a 256 -c theorem-node-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256

# 5. Verify against integrity.json
python3 -c "
import json, hashlib
integrity = json.load(open('releases/v0.5.0/integrity.json'))
# ... verify hashes match
"
```

### Installation (installer-capable rootfs)

```bash
# Fetch the installer-capable rootfs artifact, not ISO/memstick media
curl -LO <release-url>/theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz
curl -LO <release-url>/theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256
shasum -a 256 -c theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz.sha256

tar xzf theoremos-v0.5.0-x86_64-unknown-freebsd.tar.gz
cd theoremos-v0.5.0-x86_64-unknown-freebsd

# Optional first-boot provisioning
cp etc/theorem/first-boot.conf.example ./first-boot.conf
$EDITOR ./first-boot.conf

# Non-interactive install onto a prepared FreeBSD host
sh install.sh
```

### Health Check

```bash
# Liveness
curl -f http://localhost:3456/health

# Readiness + state
curl -s http://localhost:3456/status | python3 -m json.tool
```

## What We Commit To Fixing Before 0.6.0

These are the gaps between "platform preview" and "production ready":

1. **Direct-write disk image contract** — publish a machine-readable direct-write
   artifact or keep installer-rootfs as the only automation contract
2. **Upgrade contract** — documented, machine-executable N-1 → N path
3. **Rollback contract** — documented rollback with ZFS snapshot integration
4. **Configuration contract** — frozen config file format and location
5. **Release validator in CI** — `scripts/validate-release.sh` runs before
   any publication

## Versioning

theoremOS uses `0.x.y` versioning during the platform preview stage.

- `0.x.0` — new capabilities, may include breaking changes
- `0.x.y` — patch releases, no breaking changes within a minor line

When the consumer contract reaches **production ready**, versioning will
move to `1.0.0` with standard semver guarantees.
