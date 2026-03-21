# theorem-node — Quick Start

This guide is for a fast developer or lab install from a published release.
For a production enclave host, use the operator guide in
`docs/operator-guide/02-deployment.md`.

## Download

Grab the latest release for your platform from [GitHub Releases](https://github.com/Theorem-Governance/theorem-releases/releases):

```bash
# Example for x86_64 Linux — replace VERSION with the tag (e.g. v2.0.0)
VERSION="v2.0.0"
curl -LO "https://github.com/Theorem-Governance/theorem-releases/releases/download/${VERSION}/theorem-node-${VERSION}-x86_64-unknown-linux-gnu.tar.gz"
tar xzf theorem-node-*.tar.gz
cd theorem-node-*/
```

Release asset names are stable:

- `theorem-node-<tag>-<target>.tar.gz`
- `theorem-node-<tag>-<target>.tar.gz.sha256`
- `theorem-node-<tag>.cdx.xml` or equivalent CycloneDX SBOM attachment

Use an explicit tag in automation. Do not automate against the GitHub
"latest" pointer unless you have separately accepted the risk of preemption by
newer releases.

Verify the checksum:

```bash
shasum -a 256 -c theorem-node-*.tar.gz.sha256
```

## Bootstrap

Compute the spec hash from the governance specification document provided with your license:

```bash
SPEC_HASH=$(shasum -a 256 theorem-spec.md | awk '{print $1}')
```

Initialize a governance instance:

```bash
./theorem-node --db theorem.db bootstrap \
  --spec-hash "$SPEC_HASH"
```

Optional bootstrap flags (have sensible defaults for standalone operation):

| Flag | Default | Description |
|---|---|---|
| `--artifact-url` | `https://localhost/artifact` | Verification artifact location |
| `--log-endpoint` | `https://localhost/log` | Attestation log endpoint |

A signing key reference is derived automatically from the spec hash. Attestation integrity proofs use SHA-256 content hashing.

## Configure

Set environment variables (e.g. in a `.env` file sourced before running):

| Variable | Required | Default | Description |
|---|---|---|---|
| `THEOREM_COMMITMENT_SECRET` | **Yes** | — | HMAC secret for intent tokens (min 32 chars) |
| `THEOREM_ENFORCEMENT_MODE` | No | `advisory` | `advisory`, `warn`, or `enforce` |
| `THEOREM_DRIFT_THRESHOLD` | No | `0.15` | Drift tolerance (0.0–1.0) |
| `THEOREM_COMMITMENT_TOKEN_TTL_HOURS` | No | `24` | Token expiry in hours |
| `THEOREM_PEER_WITNESSES` | No | — | Comma-separated witness peer URLs |
| `THEOREM_WITNESS_TOKEN` | No | — | Bearer token for witness-authenticated routes and `/ready`; omit only for basic local development |
| `THEOREM_LOG_LEVEL` | No | `info` | Log level |

## Run

```bash
export THEOREM_COMMITMENT_SECRET="$(openssl rand -hex 32)"
./theorem-node --db theorem.db serve --port 3170
```

The API will be available at `http://localhost:3170`.

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

## Available platforms

- `x86_64-unknown-linux-gnu`
- `aarch64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

## SBOM

A CycloneDX SBOM is attached to each release for supply chain review.

## Release Contract

The supported release contract for automation, asset naming, targets, checksums,
and SBOM expectations is documented in `docs/release-contract.md`.
