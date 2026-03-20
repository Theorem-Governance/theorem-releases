# Theorem

Theorem is a governance runtime built around a property-proven, deterministic
kernel.

Its product goal is not only to record governance, but to become the trusted
state machine for authority, revocation, release verification, and fail-closed
governed execution.

## Workspace Crates

| Crate | Description |
|---|---|
| [`theorem-kernel`](theorem-kernel/) | Ring 0 governance kernel: primitives, enforcement, audit, verifier, write path |
| [`theorem-storage-sqlite`](theorem-storage-sqlite/) | SQLite storage adapter for kernel state with in-memory cache |
| [`theorem-node`](theorem-node/) | Main binary: CLI + HTTP API + attestation finalization + witness publication |
| [`theorem-witness`](theorem-witness/) | Append-only witness log model and peer verification logic |
| [`theorem-witness-storage`](theorem-witness-storage/) | SQLite-backed witness log implementation |
| [`theorem-test-support`](theorem-test-support/) | Test-only in-memory support state and helpers |
| [`theorem-bus`](theorem-bus/) | Ring 3 governance bus translating agent protocol into node ceremonies |
| [`theorem-bus-client`](theorem-bus-client/) | Typed Rust client SDK for the governance bus |
| [`theorem-gateway`](theorem-gateway/) | HTTP sidecar exposing a simpler external API over the bus |
| [`theorem-supervisor`](theorem-supervisor/) | Governed process supervisor for Ring 3 agent execution |
| [`theorem-govagent`](theorem-govagent/) | Governed autonomous worker exercising the runtime under real load |
| [`theorem-dashboard`](theorem-dashboard/) | Monitoring and observability dashboard proxy |

## Architecture

Theorem is intentionally layered:

- `theorem-kernel` is pure logic. It does not perform I/O and emits sealed
  record batches after enforcement.
- `theorem-storage-sqlite` persists those batches atomically and exposes the
  state traits the kernel reads from.
- `theorem-node` is the canonical runtime surface for bootstrap, audit,
  verification, admission, capability issuance, export, and rebuild.
- `theorem-witness` and `theorem-witness-storage` provide independent
  attestation witnessing.
- `theorem-bus`, `theorem-gateway`, `theorem-supervisor`, and
  `theorem-govagent` form the Ring 3 runtime that consumes theorem-issued
  authority.

```text
Ring 0
  theorem-kernel

Persistence and witnessing
  theorem-storage-sqlite
  theorem-witness
  theorem-witness-storage

Canonical node runtime
  theorem-node

Ring 3 governed execution
  theorem-bus
  theorem-bus-client
  theorem-gateway
  theorem-supervisor
  theorem-govagent

Observability
  theorem-dashboard
```

## Public Contracts

- Fast developer install: [`docs/quickstart.md`](docs/quickstart.md)
- Production deployment and operations:
  [`docs/operator-guide/`](docs/operator-guide/)
- Release asset and checksum contract:
  [`docs/release-contract.md`](docs/release-contract.md)
- Stable documented HTTP subset:
  [`openapi.yaml`](openapi.yaml)

Note: the node exposes additional admin and runtime-internal routes beyond the
current OpenAPI file. The OpenAPI contract documents the stable public subset,
not every live adapter-facing route.

## Quick Start

```bash
cargo build --workspace
./target/debug/theorem-node bootstrap --spec-hash <hash>
./target/debug/theorem-node status
./target/debug/theorem-node audit
./target/debug/theorem-node verify
./target/debug/theorem-node serve --port 3170
```

For release-only installs, checksum verification, systemd deployment, backup,
and recovery, use the operator guide instead of the developer quickstart.

## Specification

The formal specification lives in [`theorem-spec.md`](theorem-spec.md) with its
hash in [`theorem-spec.sha256`](theorem-spec.sha256). Formal modeling lives in
[`formal/`](formal/). Design decisions are recorded in [`DECISIONS.md`](DECISIONS.md).

## License

MIT -- see [LICENSE](LICENSE) for details.
