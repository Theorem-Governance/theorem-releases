# theorem-node — Quick Start

This guide is for a fast developer or lab install from a published release.
Expect a basic bring-up to take about 5-10 minutes once you have the release
artifacts and concrete bootstrap inputs.

For production deployment, recovery, conformance, and staged rollout
expectations, see `docs/release-contract.md`.

## What This Guide Gets You

This quickstart is for one thing: getting `theorem-node` downloaded,
bootstrapped, serving traffic, and answering the core runtime discovery and
health surfaces.

It is not the full production contract. The release contract covers recovery,
release integrity, automation, conformance, compatibility, and staged rollout
expectations.

## Before You Start

Have these ready:

- a published release tag such as `v0.5.0`
- the release tarball and checksum for your platform
- the matching governance specification document used to compute `SPEC_HASH`
- concrete values for `ARTIFACT_URL` and `LOG_ENDPOINT`
- a local port to serve on, such as `3170`

## Download

Grab the latest release for your platform from [GitHub Releases](https://github.com/Theorem-Governance/theorem-releases/releases):

```bash
# Example for x86_64 Linux — replace VERSION with the tag (e.g. v0.5.0)
VERSION="v0.5.0"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/releases/download/${VERSION}/theorem-node-${VERSION}-x86_64-unknown-linux-gnu.tar.gz"
tar xzf theorem-node-*.tar.gz
cd theorem-node-*/
```

Release asset names are stable:

- `theorem-node-<tag>-<target>.tar.gz`
- `theorem-node-<tag>-<target>.tar.gz.sha256`
- `theorem-node-<tag>.cdx.xml` or equivalent CycloneDX SBOM attachment
- `theorem-release-integrity-<tag>.json` for `4.x` release integrity evidence

Use an explicit tag in automation. Do not automate against the GitHub
"latest" pointer unless you have separately accepted the risk of preemption by
newer releases.

Verify the checksum:

```bash
shasum -a 256 -c theorem-node-*.tar.gz.sha256
```

For `4.x`, also review the release manifest and integrity evidence before
promotion:

```text
releases/<tag>/manifest.json
releases/<tag>/integrity.json
```

If you mirror releases internally, fail closed when the public release feed is
past `mirror_expires_at` in `releases/feed.json`.

## Bootstrap

Compute the spec hash from the governance specification document provided with your license:

```bash
SPEC_HASH=$(shasum -a 256 theorem-spec.md | awk '{print $1}')
```

Initialize a governance instance:

```bash
./theorem-node --db theorem.db bootstrap \
  --spec-hash "$SPEC_HASH" \
  --artifact-url <ARTIFACT_URL> \
  --log-endpoint <LOG_ENDPOINT>
```

Use concrete deployment URLs. There are no built-in bootstrap-local defaults
for these values anymore.

A signing key reference is derived automatically from the spec hash. Attestation integrity proofs use SHA-256 content hashing.

## Configure

Set environment variables (e.g. in a `.env` file sourced before running):

| Variable | Required | Default | Description |
|---|---|---|---|
| `THEOREM_COMMITMENT_SECRET` | **Yes** | — | HMAC secret for intent tokens (min 32 chars) |
| `THEOREM_ENFORCEMENT_MODE` | No | `advisory` | `advisory`, `warn`, or `enforce` |
| `THEOREM_DRIFT_THRESHOLD` | No | `0.15` | Drift tolerance (0.0–1.0) |
| `THEOREM_COMMITMENT_TOKEN_TTL_HOURS` | No | `24` | Token expiry in hours |
| `THEOREM_WITNESS_ENDPOINTS` | No | — | Comma-separated witness peer URLs |
| `THEOREM_WITNESS_TOKEN` | No | — | Bearer token for witness-authenticated routes and `/ready`; omit only for basic local development |
| `THEOREM_LOG_LEVEL` | No | `info` | Log level |

`THEOREM_PEER_WITNESSES` is still accepted as a legacy compatibility alias, but
new deployment automation should use `THEOREM_WITNESS_ENDPOINTS`.

Minimal local example:

```bash
export THEOREM_COMMITMENT_SECRET="$(openssl rand -hex 32)"
export THEOREM_LOG_LEVEL=info
```

## Run

```bash
./theorem-node --db theorem.db serve --port 3170
```

The API will be available at `http://localhost:3170`.

### Runtime Discovery

The running `4.x` server now publishes machine-readable discovery surfaces:

```bash
curl http://localhost:3170/
curl http://localhost:3170/openapi.json
curl http://localhost:3170/capabilities
curl http://localhost:3170/capability-negotiation
curl http://localhost:3170/auth/discovery
```

`/auth/discovery` names the current bearer-auth header and environment variable
used for protected routes. `/openapi.json` and `/openapi.yaml` are the live API
schema surfaces.

## Verify it's working

```bash
curl http://localhost:3170/health
curl -i http://localhost:3170/ready
./theorem-node --db theorem.db status
```

`/health` is the canonical liveness probe and returns HTTP `200` with a minimal
JSON status body whenever the process is up.

`/ready` is the readiness probe. It returns HTTP `200` only when the instance
is bootstrapped and both `THEOREM_COMMITMENT_SECRET` and
`THEOREM_WITNESS_TOKEN` are configured. In a minimal local setup you can run
without `THEOREM_WITNESS_TOKEN`, but `/ready` will remain `503 not_ready`.

`/status` is the richer governance-state inspection command.

Protected routes such as `/organizations/events`, `/organizations/explain`, and
the GET/POST `/execution-capabilities/lifecycle` surface require:

```text
Authorization: Bearer <THEOREM_WITNESS_TOKEN>
```

## Available platforms

- `x86_64-unknown-linux-gnu`
- `aarch64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

## SBOM

A CycloneDX SBOM is attached to each release for supply chain review.

## Release Integrity

`4.x` releases also publish `releases/<tag>/integrity.json` with SBOM,
provenance, minimum-version, rollback-authorization, revocation, and
mirror-expiry evidence.

## Offline Recovery Verification

Session 8 adds an explicit recovery proof bundle for measured boot and replay.
Treat recovery as lawful only when the recovery evidence proves the machine
returned to the same authority state, not merely that it rebooted.

The recovery gate passes only when:

- measured boot binds the approved recovery image and theorem runtime
- trust-root custody signs the active recovery root
- replay and anti-replay bindings match the pre-loss authority-state digest
- the recovered machine returns to the same lawful authority state

## Release Contract

The supported release contract for automation, asset naming, targets, checksums,
and SBOM expectations is documented in `docs/release-contract.md`.

## Read Next

After basic bring-up, the most important public doc is:

- `docs/release-contract.md`
