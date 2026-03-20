# Theorem

A governance system with a property-proven, deterministic kernel.

Theorem enforces governance rules as code: entities, scopes, roles, mandates, and proof obligations are managed through a formally specified kernel with 16 enforcement rules and 22 audit criteria.

## Workspace Crates

| Crate | Description |
|---|---|
| [`theorem-kernel`](theorem-kernel/) | Ring 0 governance kernel -- pure logic, no I/O |
| [`theorem-storage-sqlite`](theorem-storage-sqlite/) | SQLite storage adapter with in-memory cache |
| [`theorem-node`](theorem-node/) | CLI binary and HTTP server |
| [`theorem-watchdog`](theorem-watchdog/) | Infrastructure monitoring |

## Architecture

The kernel is a pure function: `(input, &dyn GovernanceState) -> RecordBatch`. It has no opinions about storage, networking, or runtime. The node binary wires the kernel to SQLite persistence and exposes CLI commands and an HTTP API.

```text
theorem-node          CLI + HTTP server (binary)
  |-- theorem-kernel          pure governance logic (library)
  |-- theorem-storage-sqlite  SQLite persistence (library)
  '-- theorem-watchdog        infrastructure monitoring (library)
```

## Quick Start

```bash
cargo build --workspace
./target/debug/theorem-node bootstrap --spec-hash <hash>
./target/debug/theorem-node status
./target/debug/theorem-node audit
./target/debug/theorem-node verify
./target/debug/theorem-node serve --port 3170
```

## Specification

The formal specification lives in [`theorem-spec.md`](theorem-spec.md) with its hash in [`theorem-spec.sha256`](theorem-spec.sha256). Design decisions are recorded in [`DECISIONS.md`](DECISIONS.md).

## License

MIT -- see [LICENSE](LICENSE) for details.
