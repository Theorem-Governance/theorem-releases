# theorem-node — Quick Start

## Download

Grab the latest release for your platform from [GitHub Releases](https://github.com/Theorem-Governance/theorem-releases/releases):

```bash
# Example for x86_64 Linux
curl -LO https://github.com/Theorem-Governance/theorem-releases/releases/latest/download/theorem-node-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
tar xzf theorem-node-*.tar.gz
cd theorem-node-*/
```

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
| `THEOREM_WITNESS_TOKEN` | No | — | Bearer token for witness auth |
| `THEOREM_LOG_LEVEL` | No | `info` | Log level |

## Run

```bash
export THEOREM_COMMITMENT_SECRET="$(openssl rand -hex 32)"
./theorem-node --db theorem.db serve --port 3170
```

The API will be available at `http://localhost:3170`.

## Verify it's working

```bash
./theorem-node --db theorem.db status
```

## Available platforms

- `x86_64-unknown-linux-gnu`
- `aarch64-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `aarch64-apple-darwin`

## SBOM

A CycloneDX SBOM is attached to each release for supply chain review.
